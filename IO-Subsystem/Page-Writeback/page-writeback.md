## Page Cache Writeback机制

### Bdi的概念
Bdi的全称是Backing Device Info，用来描述非易失存储设备的相关信息，使用一个结构体 `backing_dev_info` 来表示:
```c
struct backing_dev_info {
	struct list_head bdi_list;
	unsigned long ra_pages;	/* max readahead in PAGE_SIZE units */
	unsigned long io_pages;	/* max allowed IO size */
	congested_fn *congested_fn; /* Function pointer if device is md/dm */
	void *congested_data;	/* Pointer to aux data for congested func */

	const char *name;

	struct kref refcnt;	/* Reference counter for the structure */
	unsigned int capabilities; /* Device capabilities */
	unsigned int min_ratio;
	unsigned int max_ratio, max_prop_frac;

	/*
	 * Sum of avg_write_bw of wbs with dirty inodes.  > 0 if there are
	 * any dirty wbs, which is depended upon by bdi_has_dirty().
	 */
	atomic_long_t tot_write_bandwidth; /* 表示total wb任务的bandwidth之和，带宽指单位时间写有多少次io */

	struct bdi_writeback wb;  /* the root writeback info for this bdi */
	struct list_head wb_list; /* 保存着等待处理的wb任务 */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct radix_tree_root cgwb_tree; /* radix tree of active cgroup wbs */
	struct rb_root cgwb_congested_tree; /* their congested states */
	struct mutex cgwb_release_mutex;  /* protect shutdown of wb structs */
#else
	struct bdi_writeback_congested *wb_congested;
#endif
	wait_queue_head_t wb_waitq;

	struct device *dev;
	struct device *owner;

	struct timer_list laptop_mode_wb_timer;

#ifdef CONFIG_DEBUG_FS
	struct dentry *debug_dir;
	struct dentry *debug_stats;
#endif
};
```
一般情况下，磁盘多为块设备，它的 `backing_dev_info` 的初始化在 `blk_alloc_queue_node` 函数中进行初始化，`bdi->name` 为"block"。
```c
q->backing_dev_info->ra_pages =
	(VM_MAX_READAHEAD * 1024) / PAGE_SIZE;
q->backing_dev_info->capabilities = BDI_CAP_CGROUP_WRITEBACK;
q->backing_dev_info->name = "block";
timer_setup(&q->backing_dev_info->laptop_mode_wb_timer,
		    laptop_mode_timer_fn, 0);
```

### Bdi工作的原理
一般系统对文件进行写操作的时候，会将需要写入的页暂时保存在内存当中，当满足一定条件的时候(如暂存的页达到了一定比例，定期刷写)，将暂存在内存的页一次性写入到磁盘，从而提高IO效率，加快系统的读写速度。这种将页进行缓存，然后再一次性写入的IO方式称为Buffer IO。

### Bdi设备的初始化
Bdi设备一般通过 `add_disk->device_add_disk->__device_add_disk->bdi_register_owner->bdi_register->bdi_register_va` 调用栈进行注册，初始化 `gendisk->queue->backing_dev_info` 对象，然后加入到全局变量 `bdi_list` 这个列表中。

### Writeback流程的核心数据结构
```c
struct bdi_writeback {
	struct backing_dev_info *bdi;	/* our parent bdi */

	unsigned long state;		/* Always use atomic bitops on this */
	unsigned long last_old_flush;	/* 上次刷写数据的时间，定期wb的时候用到 */

	struct list_head b_dirty;	/* 暂存dirty inode，一般mark_dirty_inode会加入到这个list */
	struct list_head b_io;		/* 用于暂存即将要被writeback处理的inode */
	struct list_head b_more_io;	/* parked for more writeback */
	struct list_head b_dirty_time;	/* 暂存在cache过期的inode */
	spinlock_t list_lock;		/* protects the b_* lists */

	struct percpu_counter stat[NR_WB_STAT_ITEMS];

	struct bdi_writeback_congested *congested;

	unsigned long bw_time_stamp;	/* last time write bw is updated */
	unsigned long dirtied_stamp;
	unsigned long written_stamp;	/* pages written at bw_time_stamp */
	unsigned long write_bandwidth;	/* 单次wb任务的带宽 */
	unsigned long avg_write_bandwidth; /* further smoothed write bw, > 0 */

	/*
	 * The base dirty throttle rate, re-calculated on every 200ms.
	 * All the bdi tasks' dirty rate will be curbed under it.
	 * @dirty_ratelimit tracks the estimated @balanced_dirty_ratelimit
	 * in small steps and is much more smooth/stable than the latter.
	 */
	unsigned long dirty_ratelimit;
	unsigned long balanced_dirty_ratelimit;

	struct fprop_local_percpu completions;
	int dirty_exceeded;
	enum wb_reason start_all_reason; /* 本次wb任务的触发reason */

	spinlock_t work_lock;		/* protects work_list & dwork scheduling */
	struct list_head work_list; /* 保存wb_writeback_work结构的list，用于处理这次wb任务下面所有的work */
	struct delayed_work dwork;	/* work item used for writeback */

	unsigned long dirty_sleep;	/* last wait */

	struct list_head bdi_node;	/* anchored at bdi->wb_list */

#ifdef CONFIG_CGROUP_WRITEBACK
	struct percpu_ref refcnt;	/* used only for !root wb's */
	struct fprop_local_percpu memcg_completions;
	struct cgroup_subsys_state *memcg_css; /* the associated memcg */
	struct cgroup_subsys_state *blkcg_css; /* and blkcg */
	struct list_head memcg_node;	/* anchored at memcg->cgwb_list */
	struct list_head blkcg_node;	/* anchored at blkcg->cgwb_list */

	union {
		struct work_struct release_work;
		struct rcu_head rcu;
	};
#endif
};

struct wb_writeback_work {
	long nr_pages;
	struct super_block *sb;
	unsigned long *older_than_this;
	enum writeback_sync_modes sync_mode;
	unsigned int tagged_writepages:1;
	unsigned int for_kupdate:1;
	unsigned int range_cyclic:1;
	unsigned int for_background:1;
	unsigned int for_sync:1;	/* sync(2) WB_SYNC_ALL writeback */
	unsigned int auto_free:1;	/* free on completion */
	enum wb_reason reason;		/* why was writeback initiated? */

	struct list_head list;		/* pending work list */
	struct wb_completion *done;	/* set if the caller waits */
};
```

### Writeback线程的初始化
在 `__device_add_disk` 函数分配 `backing_dev_info` 结构的时候，进行了初始化，注册的writeback线程函数名是 `wb_workfn` 然后将wb线程保存在 `bdi->wb->dwork` 变量当中，调用栈为: `bdi_alloc_node->bdi_init->cgwb_bdi_init->wb_init` 。

### Writeblck线程的触发
Writeback任务主要由两个函数直接完成: `balance_dirty_pages_ratelimited`
`wakeup_flusher_threads` 。
主要是分配page cache的时候check一下：
`balance_dirty_pages_ratelimited` : 在 `generic_perform_write` 每次进行写操作的时候，都会检查一下是否需要writeback。
`wakeup_flusher_threads` : 一般是间隔性的触发，定时writeback。

### wakeup_flusher_threads的分析
**wakeup_flusher_threads函数:** 遍历当前的bdi_list所有的bdi设备，然后对这些bdi设备执行一次flush的操作。 `reason` 代表writeback触发的原因，大部分情况下是 `WB_REASON_SYNC` 。
```c
enum wb_reason {
	WB_REASON_BACKGROUND,
	WB_REASON_VMSCAN,
	WB_REASON_SYNC,
	WB_REASON_PERIODIC,
	WB_REASON_LAPTOP_TIMER,
	WB_REASON_FREE_MORE_MEM,
	WB_REASON_FS_FREE_SPACE,
	/*
	 * There is no bdi forker thread any more and works are done
	 * by emergency worker, however, this is TPs userland visible
	 * and we'll be exposing exactly the same information,
	 * so it has a mismatch name.
	 */
	WB_REASON_FORKER_THREAD,

	WB_REASON_MAX,
};

void wakeup_flusher_threads(enum wb_reason reason)
{
	struct backing_dev_info *bdi;

	/*
	 * If we are expecting writeback progress we must submit plugged IO.
	 */
	if (blk_needs_flush_plug(current))
		blk_schedule_flush_plug(current);

	rcu_read_lock();
	list_for_each_entry_rcu(bdi, &bdi_list, bdi_list)
		__wakeup_flusher_threads_bdi(bdi, reason); /* 遍历当前的bdi_list所有的bdi设备，然后对这些bdi设备执行一次flush的操作 */
	rcu_read_unlock();
}
```
**wakeup_flusher_threads函数:** 先判断一下是否有dirty的IO，然后遍历 `bdi->wb_list` 下面的 `bdi_writeback` 对象，然后开始进行wb。
```c
static void __wakeup_flusher_threads_bdi(struct backing_dev_info *bdi,
					 enum wb_reason reason)
{
	struct bdi_writeback *wb;

	if (!bdi_has_dirty_io(bdi)) /* 先判断一下是否有dirty的writeback任务 */
		return;

	list_for_each_entry_rcu(wb, &bdi->wb_list, bdi_node)
		wb_start_writeback(wb, reason);
}
```
这里有两个问题:
>1. 怎么样的wb任务是dirty的？
>答: bdi_has_dirty_io 函数是根据bdi->tot_write_bandwidth的值去判断是否有dirty的wb的任务，而这个值则是在queue_io函数的时候改变，queue_io函数将需要被清理的inode移动到wb->b_io，同时增加bdi->tot_write_bandwidth的值，因此如果bdi->tot_write_bandwidth不为0，那么表示queue_io函数还需要继续处理，那么表示还有io没有处理完成，即dirty。
>2. bdi->wb_list里面是怎么添加的？
>答: 当一个bdi设备初始化的时候，就会将一个bdi_writeback加入到bdi->wb_list中。这个bdi_writeback的任务是调度writeback_work。

**wb_start_writeback函数:** 先判断保证同一个wb任务，只能必须先等之前的运行完毕，才能执行下一次。最后通过 `wb_wakeup` 函数实现wb。
```c
static void wb_start_writeback(struct bdi_writeback *wb, enum wb_reason reason)
{
	/* 通过判断wb-state是否设置WB_has_dirty_io这个flag来实现 */
	if (!wb_has_dirty_io(wb))
		return;
	/*
     * 先判断保证同一个wb任务，但是对于这一个wb，必须保证所有的dirty page都刷写回去之后，才会重新置0
     * 与WB_writeback_running标志不同的地方是，可能需要多次执行wb的函数才可以全部dirty page都写回磁盘，
     * 因此WB_writeback_running设置多次，所有的页写回后才会充值WB_start_all标志
     **/
	if (test_bit(WB_start_all, &wb->state) ||
	    test_and_set_bit(WB_start_all, &wb->state))
		return;

	wb->start_all_reason = reason;
	wb_wakeup(wb);
}
```
> 问题: 在哪里清除 `WB_start_all` 这个标志?
> 答案: `wb_do_writeback->wb_check_start_all` 清除标志

**wb_wakeup函数:** 判断传入的wb任务是否已经register，如果是就执行，如果不是就等待。
```c
static void wb_wakeup(struct bdi_writeback *wb)
{
	spin_lock_bh(&wb->work_lock);
	if (test_bit(WB_registered, &wb->state))
		mod_delayed_work(bdi_wq, &wb->dwork, 0); // 这里delay是0，代表马上执行注册的wb函数wb_workfn
	spin_unlock_bh(&wb->work_lock);
}
```

**wb_workfn函数:** 遍历work_list所有的wb任务，然后执行wb_do_writeback函数。
```c
void wb_workfn(struct work_struct *work)
{
	struct bdi_writeback *wb = container_of(to_delayed_work(work),
						struct bdi_writeback, dwork);
	long pages_written;

	set_worker_desc("flush-%s", dev_name(wb->bdi->dev));
	current->flags |= PF_SWAPWRITE;

	if (likely(!current_is_workqueue_rescuer() ||
		   !test_bit(WB_registered, &wb->state))) {
		/*
		 * The normal path.  Keep writing back @wb until its
		 * work_list is empty.  Note that this path is also taken
		 * if @wb is shutting down even when we're running off the
		 * rescuer as work_list needs to be drained.
		 */
		do {
			pages_written = wb_do_writeback(wb);
			trace_writeback_pages_written(pages_written);
		} while (!list_empty(&wb->work_list)); // 遍历work_list所有的wb任务，然后执行wb_do_writeback函数
	} else {
		/*
		 * bdi_wq can't get enough workers and we're running off
		 * the emergency worker.  Don't hog it.  Hopefully, 1024 is
		 * enough for efficient IO.
		 */
		pages_written = writeback_inodes_wb(wb, 1024,
						    WB_REASON_FORKER_THREAD); // 如果没有足够的worker去处理writeback，那么就同步处理。
		trace_writeback_pages_written(pages_written);
	}

	if (!list_empty(&wb->work_list))
		wb_wakeup(wb); // 如果没有全部处理完，再处理一次
	else if (wb_has_dirty_io(wb) && dirty_writeback_interval)
		wb_wakeup_delayed(wb); // 如果还有dirty的IO，延迟一阵子再执行一次

	// 顺利执行完毕，退出这次的线程
	current->flags &= ~PF_SWAPWRITE;
}
```

**wb_do_writeback函数:** 遍历work_list所有的wb任务，然后执行 `wb_writeback` 处理主要wb任务。
```c
static long wb_do_writeback(struct bdi_writeback *wb)
{
	struct wb_writeback_work *work;
	long wrote = 0;

	set_bit(WB_writeback_running, &wb->state); // 设置状态为正在运行

	/**
	 * 执行force wb的时候，执行这个while，将任务加入到list，然后执行
	 * 
	 * 但是一般情况下，都是执行 固定间隔wb和后台wb
	 */
	while ((work = get_next_work_item(wb)) != NULL) { // 遍历这个wb人物里面的所有wb_writeback_work，然后写回
		trace_writeback_exec(wb, work);
		wrote += wb_writeback(wb, work); /* 核心wb函数 */
		finish_writeback_work(wb, work);
	}

	/*
	 * Check for a flush-everything request
	 */
	wrote += wb_check_start_all(wb); /* 检查是否全部的dirty的数据已经写入 */

	/*
	 * Check for periodic writeback, kupdated() style
	 */
	wrote += wb_check_old_data_flush(wb); /* 固定间隔，定期进行writeback */
	wrote += wb_check_background_flush(wb); /* 脏页达到一定比例, 后台writeback */
	clear_bit(WB_writeback_running, &wb->state); // 清除正在运行的状态

	return wrote;
}
```
> 问题1: force wb是怎么触发的
> 答：由用户将work加入到链表，然后进行触发。


**wb_writeback函数:** writeback机制的核心函数，主要步骤是将dirty inode，expired inode都迁移到一个list里面，然后进行处理。
```c
static long wb_writeback(struct bdi_writeback *wb,
			 struct wb_writeback_work *work)
{
	unsigned long wb_start = jiffies;
	long nr_pages = work->nr_pages;
	unsigned long oldest_jif;
	struct inode *inode;
	long progress;
	struct blk_plug plug;

	oldest_jif = jiffies;
	work->older_than_this = &oldest_jif;

	blk_start_plug(&plug);
	spin_lock(&wb->list_lock);
	for (;;) {
		/*
		 * Stop writeback when nr_pages has been consumed
		 */
		if (work->nr_pages <= 0)
			break;

		/*
		 * Background writeout and kupdate-style writeback may
		 * run forever. Stop them if there is other work to do
		 * so that e.g. sync can proceed. They'll be restarted
		 * after the other works are all done.
		 */
		if ((work->for_background || work->for_kupdate) &&
		    !list_empty(&wb->work_list))
			break;

		/*
		 * For background writeout, stop when we are below the
		 * background dirty threshold
		 */
		if (work->for_background && !wb_over_bg_thresh(wb))
			break;

		/*
		 * Kupdate and background works are special and we want to
		 * include all inodes that need writing. Livelock avoidance is
		 * handled by these works yielding to any other work so we are
		 * safe.
		 */
		if (work->for_kupdate) {
			oldest_jif = jiffies -
				msecs_to_jiffies(dirty_expire_interval * 10);
		} else if (work->for_background)
			oldest_jif = jiffies; // 获取当前时间

		trace_writeback_start(wb, work);
		if (list_empty(&wb->b_io))
			queue_io(wb, work); // 将需要被wb清理的inode都移动到wb->b_io中，设置一个wb是dirty，然后根据wb->tot_bandwidth增加wb->bdi->tot_bandwidth
		if (work->sb)
			progress = writeback_sb_inodes(work->sb, wb, work); // 如果含有superblock，那就执行这个，大部分执行这个
		else
			progress = __writeback_inodes_wb(wb, work);
		trace_writeback_written(wb, work);

		wb_update_bandwidth(wb, wb_start); // 更新处理的带宽，更新dirty io数目等

		/*
		 * Did we write something? Try for more
		 *
		 * Dirty inodes are moved to b_io for writeback in batches.
		 * The completion of the current batch does not necessarily
		 * mean the overall work is done. So we keep looping as long
		 * as made some progress on cleaning pages or inodes.
		 */
		if (progress)
			continue;
		/*
		 * No more inodes for IO, bail
		 */
		if (list_empty(&wb->b_more_io))
			break;
		/*
		 * Nothing written. Wait for some inode to
		 * become available for writeback. Otherwise
		 * we'll just busyloop.
		 */
		trace_writeback_wait(wb, work);
		inode = wb_inode(wb->b_more_io.prev);
		spin_lock(&inode->i_lock);
		spin_unlock(&wb->list_lock);
		/* This function drops i_lock... */
		inode_sleep_on_writeback(inode);
		spin_lock(&wb->list_lock);
	} // end for
	spin_unlock(&wb->list_lock);
	blk_finish_plug(&plug);

	return nr_pages - work->nr_pages; // 表示写了多少个页
}
```
**writeback_sb_inodes函数:** 将inode的dirty pages写入到磁盘当中。
```c
static long writeback_sb_inodes(struct super_block *sb,
				struct bdi_writeback *wb,
				struct wb_writeback_work *work)
{
	struct writeback_control wbc = {
		.sync_mode		= work->sync_mode,
		.tagged_writepages	= work->tagged_writepages,
		.for_kupdate		= work->for_kupdate,
		.for_background		= work->for_background,
		.for_sync		= work->for_sync,
		.range_cyclic		= work->range_cyclic,
		.range_start		= 0,
		.range_end		= LLONG_MAX,
	};
	unsigned long start_time = jiffies;
	long write_chunk;
	long wrote = 0;  /* count both pages and inodes */

	while (!list_empty(&wb->b_io)) { // 遍历wb->b_io里面的所有inode
		struct inode *inode = wb_inode(wb->b_io.prev); // 从list中取出一个inode
		struct bdi_writeback *tmp_wb;

		if (inode->i_sb != sb) {
			if (work->sb) {
				/*
				 * We only want to write back data for this
				 * superblock, move all inodes not belonging
				 * to it back onto the dirty list.
				 */
				redirty_tail(inode, wb);
				continue;
			}

			/*
			 * The inode belongs to a different superblock.
			 * Bounce back to the caller to unpin this and
			 * pin the next superblock.
			 */
			break;
		}

		/*
		 * Don't bother with new inodes or inodes being freed, first
		 * kind does not need periodic writeout yet, and for the latter
		 * kind writeout is handled by the freer.
		 */
		spin_lock(&inode->i_lock);
		if (inode->i_state & (I_NEW | I_FREEING | I_WILL_FREE)) {
			spin_unlock(&inode->i_lock);
			redirty_tail(inode, wb);
			continue;
		}
		if ((inode->i_state & I_SYNC) && wbc.sync_mode != WB_SYNC_ALL) { /* 这个情况一般是 内存不够刷写的情况出现，因为wbc.sync_mode = WB_SYNC_NONE */
			/*
			 * If this inode is locked for writeback and we are not
			 * doing writeback-for-data-integrity, move it to
			 * b_more_io so that writeback can proceed with the
			 * other inodes on s_io.
			 *
			 * We'll have another go at writing back this inode
			 * when we completed a full scan of b_io.
			 */
			spin_unlock(&inode->i_lock);
			requeue_io(inode, wb); // 将这个inode迁移到wb->b_more_io，下次再执行，因为模式是WB_SYNC_NONE，不等待SYNC的完成
			trace_writeback_sb_inodes_requeue(inode);
			continue;
		}
		spin_unlock(&wb->list_lock);

		/*
		 * We already requeued the inode if it had I_SYNC set and we
		 * are doing WB_SYNC_NONE writeback. So this catches only the
		 * WB_SYNC_ALL case.
		 */
		if (inode->i_state & I_SYNC) { /* 对于SYNC模式的inode和SYNC的wb类型 */
			/* Wait for I_SYNC. This function drops i_lock... */
			inode_sleep_on_writeback(inode);
			/* Inode may be gone, start again */
			spin_lock(&wb->list_lock);
			continue;
		}
		inode->i_state |= I_SYNC;
		wbc_attach_and_unlock_inode(&wbc, inode); // 对于inode进行解锁

		write_chunk = writeback_chunk_size(wb, work); // 计算需要写多少数据
		wbc.nr_to_write = write_chunk; // 一般是4096
		wbc.pages_skipped = 0;
		/*
		 * We use I_SYNC to pin the inode in memory. While it is set
		 * evict_inode() will wait so the inode cannot be freed.
		 */
		__writeback_single_inode(inode, &wbc); // 写入将inode的数据磁盘，会使用到vfs的writepages函数

		wbc_detach_inode(&wbc);
		work->nr_pages -= write_chunk - wbc.nr_to_write; // 记录写了多少页
		wrote += write_chunk - wbc.nr_to_write;

		if (need_resched()) {
			/*
			 * We're trying to balance between building up a nice
			 * long list of IOs to improve our merge rate, and
			 * getting those IOs out quickly for anyone throttling
			 * in balance_dirty_pages().  cond_resched() doesn't
			 * unplug, so get our IOs out the door before we
			 * give up the CPU.
			 */
			blk_flush_plug(current);
			cond_resched();
		}

		/*
		 * Requeue @inode if still dirty.  Be careful as @inode may
		 * have been switched to another wb in the meantime.
		 */
		tmp_wb = inode_to_wb_and_lock_list(inode); // 给wb加锁
		spin_lock(&inode->i_lock);
		if (!(inode->i_state & I_DIRTY_ALL))
			wrote++;
		requeue_inode(inode, tmp_wb, &wbc);
		inode_sync_complete(inode);
		spin_unlock(&inode->i_lock);

		if (unlikely(tmp_wb != wb)) {
			spin_unlock(&tmp_wb->list_lock);
			spin_lock(&wb->list_lock);
		}

		/*
		 * bail out to wb_writeback() often enough to check
		 * background threshold and other termination conditions.
		 */
		if (wrote) {
			if (time_is_before_jiffies(start_time + HZ / 10UL))
				break;
			if (work->nr_pages <= 0)
				break;
		}
	}
	return wrote;
}
```