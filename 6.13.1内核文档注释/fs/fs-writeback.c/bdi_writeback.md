**`bdi_writeback` 结构体**

`bdi_writeback` 结构体也定义在 `fs/fs-writeback.c` 文件中。  它代表一个 **Backing Device Info (BDI) 的 writeback 控制器**。  每个 BDI (例如，每个块设备或每个 backing_dev_info 实例) 都有一个关联的 `bdi_writeback` 结构体，用于管理和控制该 BDI 上的 writeback 活动。

```c
struct bdi_writeback {
	struct backing_dev_info *bdi;	/* 我们的父 BDI */

	unsigned long state;		/* 始终使用原子位操作 */
	unsigned long last_old_flush;	/* 上次旧数据 flush 的时间 */

	struct list_head b_dirty;	/* 脏 inode 链表 */
	struct list_head b_io;		/* 停放等待 writeback 的 inode 链表 */
	struct list_head b_more_io;	/* 停放等待更多 writeback 的 inode 链表 */
	struct list_head b_dirty_time;	/* 时间戳是脏的 inode 链表 */
	spinlock_t list_lock;		/* 保护 b_* 链表的自旋锁 */

	atomic_t writeback_inodes;	/* 正在进行 writeback 的 inode 数量 */
	struct percpu_counter stat[NR_WB_STAT_ITEMS]; // Per-CPU 计数器数组，用于统计 writeback 信息

	unsigned long bw_time_stamp;	/* 上次更新 write 带宽的时间戳 */
	unsigned long dirtied_stamp;    /* 上次脏页统计的时间戳 */
	unsigned long written_stamp;	/* 在 bw_time_stamp 时刻写入的页数 */
	unsigned long write_bandwidth;	/* 估计的 write 带宽 (bytes/秒) */
	unsigned long avg_write_bandwidth; /* 进一步平滑的 write 带宽，> 0 */

	/*
	 * 基本脏页节流速率，每 200ms 重新计算一次。
	 * 所有 bdi 任务的脏页速率都将受到它的限制。
	 * @dirty_ratelimit 以小步跟踪估计的 @balanced_dirty_ratelimit，
	 * 并且比后者更平滑/稳定。
	 */
	unsigned long dirty_ratelimit;
	unsigned long balanced_dirty_ratelimit;

	struct fprop_local_percpu completions; // Per-CPU 完成计数器，用于跟踪 writeback 完成事件
	int dirty_exceeded; // 脏页是否超过阈值标志
	enum wb_reason start_all_reason; // 启动所有 writeback 的原因

	spinlock_t work_lock;		/* 保护 work_list 和 dwork 调度的自旋锁 */
	struct list_head work_list; // 工作队列链表，用于异步 writeback 工作项
	struct delayed_work dwork;	/* 用于 writeback 的延迟工作项 */
	struct delayed_work bw_dwork;	/* 用于带宽估计的延迟工作项 */

	struct list_head bdi_node;	/* 锚定在 bdi->wb_list 链表上 */

#ifdef CONFIG_CGROUP_WRITEBACK
	struct percpu_ref refcnt;	/* 仅用于非 root wb */
	struct fprop_local_percpu memcg_completions; // Per-CPU 完成计数器，用于 memcg writeback 完成事件
	struct cgroup_subsys_state *memcg_css; /* 关联的 memcg cgroup 子系统状态 */
	struct cgroup_subsys_state *blkcg_css; /* 关联的 blkcg cgroup 子系统状态 */
	struct list_head memcg_node;	/* 锚定在 memcg->cgwb_list 链表上 */
	struct list_head blkcg_node;	/* 锚定在 blkcg->cgwb_list 链表上 */
	struct list_head b_attached;	/* 附加的 inode 链表，受 list_lock 保护 */
	struct list_head offline_node;	/* 锚定在 offline_cgwbs 链表上 */

	union {
		struct work_struct release_work; // 释放工作项
		struct rcu_head rcu; // RCU 回调头部
	};
#endif
};
```

**`bdi_writeback` 结构体成员详细解析:**

* **`struct backing_dev_info *bdi;`**:  **父 BDI 指针。**  指向关联的 `backing_dev_info` 结构体。  `backing_dev_info` 结构体描述了块设备或 backing storage 的属性和特性。  `bdi_writeback` 结构体是 `backing_dev_info` 的一部分，用于管理该 BDI 上的 writeback 活动。

* **`unsigned long state;`**:  **状态标志。**  使用原子位操作访问和修改。  用于记录 `bdi_writeback` 的各种状态，例如是否正在运行 writeback 工作队列、是否需要进行带宽估计等。  具体状态标志定义可以参考内核代码。

* **`unsigned long last_old_flush;`**:  **上次旧数据 flush 的时间。**  用于跟踪上次执行旧数据 flush 操作的时间戳。  可能与 writeback 节流或优先级控制有关。

* **Inode 链表 (Inode Lists):**  以下 `list_head` 类型的成员用于维护不同状态的 inode 链表，用于 writeback 调度和管理。  `list_lock` 自旋锁用于保护这些链表的并发访问。

    * **`struct list_head b_dirty;`**:  **脏 inode 链表。**  链表中的 inode 都包含脏页，需要进行 writeback。  Inode 在被标记为脏时会被添加到这个链表，在 writeback 完成后会被移除。

    * **`struct list_head b_io;`**:  **I/O 进行中 inode 链表。**  链表中的 inode 正在进行 writeback 操作。  Inode 在开始 writeback 时会从 `b_dirty` 链表移动到 `b_io` 链表，在 writeback 完成后会被移除。

    * **`struct list_head b_more_io;`**:  **需要更多 I/O 的 inode 链表。**  在某些情况下，inode 可能需要进行多轮 writeback 操作。  例如，当 inode 的脏页数量非常大，或者 writeback 操作被中断时，inode 可能会被添加到 `b_more_io` 链表，等待后续的 writeback 调度。

    * **`struct list_head b_dirty_time;`**:  **时间戳脏 inode 链表。**  链表中的 inode 的时间戳 (例如，atime, mtime, ctime) 是脏的，需要写回。  与数据页脏 inode 链表 `b_dirty` 分开管理，可能用于优化时间戳更新的 writeback 策略。

    * **`spinlock_t list_lock;`**:  **链表锁。**  自旋锁，用于保护 `b_dirty`, `b_io`, `b_more_io`, `b_dirty_time`, `b_attached` 等 inode 链表的并发访问。

* **`atomic_t writeback_inodes;`**:  **正在进行 writeback 的 inode 数量。**  原子计数器，记录当前正在进行 writeback 操作的 inode 数量。  用于统计和监控 writeback 活动。

* **`struct percpu_counter stat[NR_WB_STAT_ITEMS];`**:  **Per-CPU 计数器数组。**  用于统计各种 writeback 相关的性能指标，例如，写入的页数、写入的字节数、完成的 I/O 操作数等。  Per-CPU 计数器可以减少多核系统上的锁竞争，提高统计效率。  `NR_WB_STAT_ITEMS` 定义了统计项的数量。

* **带宽估计 (Bandwidth Estimation):**  以下成员用于 writeback 带宽估计和节流。

    * **`unsigned long bw_time_stamp;`**:  **上次更新 write 带宽的时间戳。**  记录上次更新 `write_bandwidth` 和 `avg_write_bandwidth` 的时间戳。  用于周期性地更新带宽估计值。

    * **`unsigned long dirtied_stamp;`**:  **上次脏页统计的时间戳。**  记录上次统计脏页数量的时间戳。  可能用于计算脏页产生速率，用于动态调整 writeback 节流参数。

    * **`unsigned long written_stamp;`**:  **在 `bw_time_stamp` 时刻写入的页数。**  记录在 `bw_time_stamp` 时刻已经写入的页数。  与 `bw_time_stamp` 和当前写入页数一起，用于计算 write 带宽。

    * **`unsigned long write_bandwidth;`**:  **估计的 write 带宽 (bytes/秒)。**  基于历史 I/O 统计信息估计的当前 write 带宽。  用于 writeback 节流，防止 writeback 操作过度消耗 I/O 资源。

    * **`unsigned long avg_write_bandwidth;`**:  **进一步平滑的 write 带宽。**  对 `write_bandwidth` 进行进一步平滑处理，得到更稳定的平均 write 带宽。  用于更平稳的 writeback 节流控制。

    * **`unsigned long dirty_ratelimit;`**:  **脏页速率限制。**  基本脏页节流速率，每 200ms 重新计算一次。  用于限制脏页的产生速率，防止脏页积累过快，导致内存压力。

    * **`unsigned long balanced_dirty_ratelimit;`**:  **平衡的脏页速率限制。**  目标脏页速率限制，用于动态调整 `dirty_ratelimit`。  `dirty_ratelimit` 会逐步逼近 `balanced_dirty_ratelimit`，实现更平稳的节流控制。

* **`struct fprop_local_percpu completions;`**:  **Per-CPU 完成计数器。**  用于跟踪 writeback 完成事件 (例如，完成的 I/O 操作数)。  与 `stat` 数组类似，使用 Per-CPU 计数器提高效率。

* **`int dirty_exceeded;`**:  **脏页超过阈值标志。**  如果脏页数量超过预设的阈值，则设置此标志。  用于触发更积极的 writeback 操作，缓解内存压力。

* **`enum wb_reason start_all_reason;`**:  **启动所有 writeback 的原因。**  记录触发全局 writeback 操作 (例如，`wb_start_background_writeback` 或 `wb_start_sync_writeback`) 的原因。  用于调试和分析 writeback 行为。  `enum wb_reason` 定义了可能的原因，例如，内存压力、周期性定时器、`sync` 系统调用等。

* **工作队列 (Workqueue):**  以下成员用于异步 writeback 工作队列和延迟工作项。

    * **`spinlock_t work_lock;`**:  **工作队列锁。**  自旋锁，用于保护 `work_list` 链表和延迟工作项 `dwork`, `bw_dwork` 的并发访问和调度。

    * **`struct list_head work_list;`**:  **工作队列链表。**  用于存放异步 writeback 工作项 (通常是 `wb_workfn` 函数)。  Writeback 守护进程 (例如，`bdi-default`) 会从 `work_list` 中取出工作项并执行。

    * **`struct delayed_work dwork;`**:  **延迟工作项 (用于 writeback)。**  用于延迟执行 writeback 工作。  例如，在后台 writeback 场景下，可以使用延迟工作项来周期性地唤醒 writeback 守护进程。

    * **`struct delayed_work bw_dwork;`**:  **延迟工作项 (用于带宽估计)。**  用于延迟执行带宽估计更新操作。  例如，可以定期使用延迟工作项来更新 `write_bandwidth` 和 `avg_write_bandwidth`。

    * **`struct list_head bdi_node;`**:  **BDI 节点。**  用于将 `bdi_writeback` 结构体添加到全局的 BDI 链表 `bdi->wb_list` 中。  用于 BDI 管理和查找。

    * ### `#ifdef CONFIG_CGROUP_WRITEBACK`部分
    * **cgroup Writeback 相关成员:**  以下成员仅在启用了 cgroup writeback 功能 (`CONFIG_CGROUP_WRITEBACK`) 时才存在。  用于支持 cgroup 级别的 writeback 隔离和资源控制。

        * **`struct percpu_ref refcnt;`**:  **Per-CPU 引用计数器。**  用于非 root `bdi_writeback` 结构体的引用计数管理。  非 root `bdi_writeback` 结构体 (例如，memcg-specific `bdi_writeback`) 需要引用计数来管理生命周期。

        * **`struct fprop_local_percpu memcg_completions;`**:  **Per-CPU 完成计数器 (memcg)。**  类似于 `completions`，但用于跟踪 memcg-specific writeback 的完成事件。

        * **`struct cgroup_subsys_state *memcg_css;`**:  **关联的 memcg cgroup 子系统状态。**  指向与此 `bdi_writeback` 关联的 memcg cgroup 子系统状态结构体。  用于 cgroup 资源控制和记账。

        * **`struct cgroup_subsys_state *blkcg_css;`**:  **关联的 blkcg cgroup 子系统状态。**  指向与此 `bdi_writeback` 关联的 blkcg cgroup 子系统状态结构体。  用于 blkcg (block cgroup) I/O 节流和资源控制。

        * **`struct list_head memcg_node;`**:  **memcg 节点。**  用于将 memcg-specific `bdi_writeback` 结构体添加到 memcg 的 `cgwb_list` 链表中。

        * **`struct list_head blkcg_node;`**:  **blkcg 节点。**  用于将 blkcg-specific `bdi_writeback` 结构体添加到 blkcg 的 `cgwb_list` 链表中。

        * **`struct list_head b_attached;`**:  **附加的 inode 链表 (cgroup)。**  类似于 `b_dirty` 等 inode 链表，但用于 cgroup writeback 场景。  链表中的 inode 是附加到当前 cgroup 的，需要进行 cgroup-aware writeback。  受 `list_lock` 保护。

        * **`struct list_head offline_node;`**:  **离线节点链表 (cgroup)。**  用于管理离线的 cgroup writeback 结构体。

        * **`union { ... }`**:  **匿名联合体。**  用于在 cgroup writeback 结构体释放时，选择使用工作队列或 RCU 回调机制来执行释放操作。
            * **`struct work_struct release_work;`**:  **释放工作项。**  用于使用工作队列机制延迟执行 `bdi_writeback` 结构体的释放操作。
            * **`struct rcu_head rcu;`**:  **RCU 回调头部。**  用于使用 RCU (Read-Copy-Update) 机制异步执行 `bdi_writeback` 结构体的释放操作。  RCU 是一种轻量级的同步机制，适用于读多写少的场景。