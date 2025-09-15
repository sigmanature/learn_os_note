struct dirty_throttle_control {
#ifdef CONFIG_CGROUP_WRITEBACK
	struct wb_domain	*dom;        // 指向写回域（writeback domain），用于cgroup隔离
	struct dirty_throttle_control *gdtc; // 在memcg的dtc中，此指针指向全局的dtc，用于对比
#endif
	struct bdi_writeback	*wb;         // 当前正在处理的写回实例 (per-bdi)
	struct fprop_local_percpu *wb_completions; // 用于跟踪IO完成情况，计算带宽

	/* 以下是全局或cgroup域的脏页状态和阈值 */
	unsigned long		avail;		/* 可用于脏页的总内存量 */
	unsigned long		dirty;		/* 当前文件脏页、正在回写的页、NFS不稳定页的总和 */
	unsigned long		thresh;		/* 硬脏页阈值 (dirty_thresh) */
	unsigned long		bg_thresh;	/* 后台回写启动阈值 (background_thresh) */
	unsigned long		limit;		/* 硬限制，几乎不会达到，用于紧急情况 */

	/* 以下是当前wb实例相关的脏页状态和阈值 */
	unsigned long		wb_dirty;	/* 这个wb实例上有多少脏页 */
	unsigned long		wb_thresh;      /* 这个wb实例的脏页硬阈值 */
	unsigned long		wb_bg_thresh;   /* 这个wb实例的后台回写阈值 */

	/* 以下是计算出的用于决策的变量 */
	unsigned long		pos_ratio;      /* 位置比例，表示脏页超出阈值的程度，值越大越严重 */
	bool			freerun;        /* 是否处于“自由运行”状态，即脏页很少，无需限速 */
	bool			dirty_exceeded; /* 脏页数量是否已超过硬阈值 */
};
