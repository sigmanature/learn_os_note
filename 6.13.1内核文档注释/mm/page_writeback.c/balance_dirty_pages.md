 好的，我们来深入分析 `balance_dirty_pages` 这个Linux内核中非常核心的I/O调节函数。它对于系统的响应性和吞吐量至关重要。

首先，我们来逐一解释代码和结构体。

---

### 第一部分：结构体 `dirty_throttle_control` 详细解释

这个结构体是 `balance_dirty_pages` 函数进行决策的核心数据集合。它整合了全局（或cgroup）和单个后备设备（backing device）的脏页信息，用于计算是否需要以及需要对当前进程进行多长时间的限速。

```c
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
```

**成员变量详解:**

*   **`dom` 和 `gdtc`**: 这两个主要用于Memory Cgroup的写回隔离。`dom` 定义了一个资源控制域，而 `gdtc` 允许一个cgroup的`dirty_throttle_control`（`mdtc`）访问全局的`dirty_throttle_control`（`gdtc`），以便在cgroup和全局两个层面同时进行限制，并取其中更严格的那个。
*   **`wb`**: `struct bdi_writeback` 的实例，代表了一个具体的写回队列。通常一个块设备（如 `/dev/sda`）对应一个 `backing_dev_info` (bdi)，而一个bdi下可以有多个 `bdi_writeback` (wb) 实例，以提高并发性。这个`wb`就是当前弄脏页面的进程所关联的那个。
*   **`wb_completions`**: 用于估算设备IO带宽。通过跟踪最近一段时间内完成的写回IO数量，内核可以知道这个设备大致的写入速度。
*   **`avail`**: 系统中可以被弄脏的内存总量。通常是 `total_memory - reserved_memory`。
*   **`dirty`**: 系统当前总的脏页数量。这是决策的最关键输入之一。
*   **`thresh`**: 脏页硬阈值。当系统脏页数量超过这个值时，弄脏页面的进程**必须**被限速（暂停）。它通常由 `vm.dirty_ratio` 或 `vm.dirty_bytes` 决定。
*   **`bg_thresh`**: 脏页后台回写阈值。当系统脏页数量超过这个值时，内核会唤醒后台的回写线程（如 `flusher` 线程）开始异步地将脏页写入磁盘。弄脏页面的进程此时**还不会**被强制暂停。它通常由 `vm.dirty_background_ratio` 或 `vm.dirty_background_bytes` 决定。
*   **`limit`**: 一个非常高的硬限制，通常是`thresh`加上一些余量。
*   **`wb_dirty`**: 只计算归属于当前`wb`的脏页数量。这用于实现设备间的公平性，防止一个慢速设备（如USB盘）上的大量脏页导致快速设备（如SSD）上的进程也被不公平地长时间限速。
*   **`wb_thresh`, `wb_bg_thresh`**: 基于`thresh`和`bg_thresh`，并根据系统中所有设备的脏页分布情况，按比例分配给当前`wb`的阈值。
*   **`pos_ratio`**: **位置比例**，这是限速算法的核心。它衡量了当前脏页数量超出后台回写阈值 (`bg_thresh`) 的严重程度。计算公式大致是 `(dirty - bg_thresh) / (thresh - bg_thresh)`。如果 `dirty < bg_thresh`，`pos_ratio` 为0。如果 `dirty >= thresh`，`pos_ratio` 为最大值。这个比例值会直接影响分配给当前任务的速率限制。
*   **`freerun`**: 一个布尔标志。如果为 `true`，表示当前脏页数量远低于阈值，系统状态良好，进程可以“自由运行”，无需任何暂停。通常在 `dirty < bg_thresh` 时为 `true`。
*   **`dirty_exceeded`**: 一个布尔标志。如果为 `true`，表示脏页数量已经超过了硬阈值 `thresh`。

---

### 第二部分：`balance_dirty_pages` 函数代码详细解释

这个函数的主体是一个 `for (;;)` 死循环，但内部有多个退出点。它的目标是：检查系统脏页状态，如果状态良好就立即退出；如果状态不佳，就计算一个暂停时间，让当前进程睡眠，醒来后再次检查，直到满足某个退出条件。

```c
static int balance_dirty_pages(struct bdi_writeback *wb,
			       unsigned long pages_dirtied, unsigned int flags)
{
	// 1. 初始化
	// gdtc_stor: 全局的 dirty_throttle_control
	// mdtc_stor: memory cgroup 的 dirty_throttle_control
	struct dirty_throttle_control gdtc_stor = { GDTC_INIT(wb) };
	struct dirty_throttle_control mdtc_stor = { MDTC_INIT(wb, &gdtc_stor) };
	// ... gdtc, mdtc 指针设置 ...
	// sdtc: Selected DTC，最终决定使用哪个（全局或cgroup）的计算结果来限速

	// 2. 无限循环，直到满足退出条件
	for (;;) {
		// 2.1 获取当前全局脏页数
		nr_dirty = global_node_page_state(NR_FILE_DIRTY);

		// 2.2 计算全局和cgroup的阈值和状态
		// 这会填充 gdtc->thresh, gdtc->bg_thresh, gdtc->freerun 等字段
		balance_domain_limits(gdtc, strictlimit);
		if (mdtc) {
			balance_domain_limits(mdtc, strictlimit);
		}
		// 我们来重点解析一下balance_domain_limits函数。这个函数会将所有和脏页平衡有关的参数
		// 全部给填充到dtc结构体里。
		// bdl函数一共顺序调用三个函数,首先是domain_dirty_avail
		// 不考虑cgroup的情况之下,dtc的dirty字段直接被设置为global_node_page_state(NR_FILE_DIRTY);
		// 从字面意思我们假定就是当前文件的脏页。avali字段被设置为global_dirtyable_memory
		// 接下来是第二个函数调用,domain_dirty_limits。先不在乎其逻辑,它就是设置thresh和bg_thresh的
		// 第三个函数调用
		// 2.3 检查是否需要启动后台回写
		// 如果不在笔记本模式，且脏页数超过后台阈值，就唤醒flusher线程
		if (!laptop_mode && nr_dirty > gdtc->bg_thresh &&
		    !writeback_in_progress(wb))
			wb_start_background_writeback(wb);

		// 2.4 【第一个退出点】检查是否处于“自由运行”状态
		// 如果全局和cgroup的脏页数都在安全范围内 (freerun为true)
		if (gdtc->freerun && (!mdtc || mdtc->freerun)) {
free_running:
			// ... 更新当前进程的脏页统计信息 ...
			// 设置一个较大的 nr_dirtied_pause，允许进程在下次检查前弄脏更多页面
			current->nr_dirtied_pause = min(intv, m_intv);
			break; // 退出循环
		}

		// 2.5 如果没有 free_running，说明脏页较多，必须确保回写正在进行
		if (unlikely(!writeback_in_progress(wb)))
			wb_start_background_writeback(wb);
		
		// ... cgroup 相关处理 ...

		// 2.6 计算 per-wb 的限速参数
		// 这会计算出 gdtc->pos_ratio
		balance_wb_limits(gdtc, strictlimit);
		if (gdtc->freerun) // 再次检查，可能在计算期间状态变化了
			goto free_running;
		sdtc = gdtc; // 默认选择全局的DTC

		if (mdtc) {
			// 如果有cgroup限制，也计算cgroup的pos_ratio
			balance_wb_limits(mdtc, strictlimit);
			if (mdtc->freerun)
				goto free_running;
			// 选择更严格的那个（pos_ratio 更小意味着限制更强，但这里代码逻辑是选pos_ratio小的，
			// 实际上pos_ratio越大限制越强，代码里是 `mdtc->pos_ratio < gdtc->pos_ratio`
			// 这是因为pos_ratio的计算方式，值越小表示离阈值越远，可分配的带宽越高，
			// 这里的比较是选择哪个域的限制更强，即哪个域的可用空间比例更小，从而导致task_ratelimit更低
			if (mdtc->pos_ratio < gdtc->pos_ratio)
				sdtc = mdtc;
		}
		
		// ... 更新带宽估算 ...

		// 2.7 计算暂停时间
		// wb->dirty_ratelimit 是设备的总IO带宽估算值（页/秒）
		dirty_ratelimit = READ_ONCE(wb->dirty_ratelimit);
		// task_ratelimit 是根据 pos_ratio 分配给当前任务的速率（页/秒）
		// pos_ratio 越大，task_ratelimit 越小，限制越强
		task_ratelimit = ((u64)dirty_ratelimit * sdtc->pos_ratio) >>
							RATELIMIT_CALC_SHIFT;
		// ... 计算最大、最小暂停时间 ...

		// 如果计算出的速率为0，说明脏页太多，必须重度惩罚
		if (unlikely(task_ratelimit == 0)) {
			pause = max_pause; // 直接暂停最大时间
			goto pause;
		}

		// 根据速率和本次弄脏的页数，计算理论上需要暂停多久
		// 时间 = 数据量 / 速率
		period = HZ * pages_dirtied / task_ratelimit;
		pause = period;
		// 减去进程“思考”的时间，做补偿
		if (current->dirty_paused_when)
			pause -= now - current->dirty_paused_when;

		// 2.8 【第二个退出点】如果计算出的暂停时间太短
		if (pause < min_pause) {
			// ... 更新统计信息，为下一次做准备 ...
			break; // 暂停时间太短，没必要睡，直接退出
		}
		// ... 调整 pause 不超过 max_pause ...

pause:
		// 2.9 【第三个退出点】异步模式
		// 如果调用者不希望阻塞，设置返回值为 -EAGAIN 并退出
		if (flags & BDP_ASYNC) {
			ret = -EAGAIN;
			break;
		}

		// 2.10 执行暂停（睡眠）
		__set_current_state(TASK_KILLABLE);
		io_schedule_timeout(pause); // 睡眠 pause 个 jiffies

		// ... 更新统计信息 ...

		// 2.11 【第四个退出点】正常暂停后的退出
		// 如果 task_ratelimit > 0，说明我们是按速率限制睡了一觉。
		// 这一觉“偿还”了弄脏页面的“债务”，可以退出循环让进程继续执行。
		if (task_ratelimit)
			break;

		// 2.12 【第五个退出点】处理卡死设备的情况
		// 如果 task_ratelimit 为 0 (说明脏页严重超标)，但当前wb的脏页数很少
		// 这通常意味着全局脏页多是由其他设备（如卡死的NFS）造成的
		// 为了避免当前进程被饿死，允许它继续执行。
		if (sdtc->wb_dirty <= wb_stat_error())
			break;

		// 2.13 【第六个退出点】进程收到致命信号
		if (fatal_signal_pending(current))
			break;

		// 如果以上退出条件都不满足（通常是 task_ratelimit=0 且 sdtc->wb_dirty 仍然很高），
		// 循环会继续，进程会再次被强制暂停 max_pause。
	}
	return ret;
}
```

---

### 第三部分：循环退出的最终条件（代码形式角度）

`balance_dirty_pages` 的 `for (;;)` 循环本质上是一个状态机：**检查 -> 决策 -> 行动(退出或睡眠) -> 重新检查**。其最终目标是让弄脏页面的进程（生产者）的速度与磁盘I/O（消费者）的速度相匹配。

循环的退出**最终**是由 `break;` 语句触发的。从代码角度看，有以下几个**直接**导致循环退出的条件：

1.  **系统脏页数量足够低 (Freerun)**
    *   **代码:** `if (gdtc->freerun && (!mdtc || mdtc->freerun)) { ... break; }`
    *   **影响变量:** `gdtc->freerun` (或 `mdtc->freerun`)。
    *   **变量值:** 这个值为 `true` 时触发退出。
    *   **深层原因:** `freerun` 变为 `true` 是因为 `balance_domain_limits()` 函数内部的检查发现，当前脏页数 `dirty` 小于后台回写阈值 `bg_thresh`。即 `global_node_page_state(NR_FILE_DIRTY) < gdtc->bg_thresh`。这是最理想的退出情况。

2.  **计算出的暂停时间过短**
    *   **代码:** `if (pause < min_pause) { ... break; }`
    *   **影响变量:** `pause`。
    *   **变量值:** `pause` 的值小于 `min_pause` (通常是1或0)。
    *   **深层原因:** `pause` 的值主要由 `task_ratelimit` 决定。如果 `task_ratelimit` 非常高（意味着系统很空闲，分配给任务的速率很高），或者 `pages_dirtied` 很小，或者进程距离上次暂停已经过了很久（`now - current->dirty_paused_when` 很大），都会导致 `pause` 变小。这本质上也是系统状态良好的体现。

3.  **异步调用模式**
    *   **代码:** `if (flags & BDP_ASYNC) { ... break; }`
    *   **影响变量:** `flags`。
    *   **变量值:** `flags` 的比特位包含 `BDP_ASYNC`。
    *   **深层原因:** 这是调用者主动选择不阻塞，希望函数立即返回当前系统是否繁忙的状态。

4.  **完成一次有效的暂停后**
    *   **代码:** `if (task_ratelimit) break;`
    *   **影响变量:** `task_ratelimit`。
    *   **变量值:** `task_ratelimit > 0`。
    *   **深层原因:** 这个 `break` 位于 `io_schedule_timeout(pause)` 之后。它的意思是：只要系统没有极端到 `task_ratelimit` 被计算为0，那么进程在睡眠了 `pause` 时间后，就认为它已经为自己弄脏的页面付出了时间的代价，可以退出循环去干活了。这是最常见的**因限速而暂停后**的退出路径。

5.  **防止被卡死的设备饿死**
    *   **代码:** `if (sdtc->wb_dirty <= wb_stat_error()) break;`
    *   **影响变量:** `sdtc->wb_dirty`。
    *   **变量值:** `sdtc->wb_dirty` 小于一个很小的误差值 `wb_stat_error()`。
    *   **深层原因:** 这个条件只有在 `task_ratelimit` 为 0 时才会被检查。`task_ratelimit` 为 0 意味着系统脏页严重超标。但如果此时当前设备 (`wb`) 上的脏页 `wb_dirty` 很少，就说明问题出在别的设备上。内核为了公平和系统的响应性，不能让这个无辜的进程一直在这里死等，所以让它退出。

6.  **进程被杀**
    *   **代码:** `if (fatal_signal_pending(current)) break;`
    *   **影响变量:** `current->pending.signal`。
    *   **变量值:** 信号集中有致命信号（如 `SIGKILL`）。
    *   **深层原因:** 进程都要死了，没必要再对它进行I/O限速了。

**总结**

循环退出的最终条件，从根本上说，取决于**系统脏页数量 (`global_node_page_state(NR_FILE_DIRTY)`) 与脏页阈值 (`bg_thresh`, `thresh`) 之间的关系**。

*   当 **`dirty < bg_thresh`** 时，会触发条件1 (`freerun = true`)，直接退出。
*   当 **`bg_thresh <= dirty < thresh`** 时，`task_ratelimit` 会被算成一个大于0的值。进程可能会被暂停一小段时间，然后通过条件4退出。
*   当 **`dirty >= thresh`** 时，`task_ratelimit` 可能会变得很小甚至为0。进程会被暂停较长时间。如果 `task_ratelimit` 仍大于0，通过条件4退出。如果 `task_ratelimit` 为0，则可能因为当前设备脏页少而通过条件5退出，或者在下一次循环中继续被暂停，直到后台回写使得 `dirty` 降下来，`task_ratelimit` 恢复为正数，最终通过条件4退出。

因此，**`task_ratelimit`** 这个变量是整个决策逻辑的核心枢纽，它的值（是否为0）和大小，直接决定了进程是继续循环（被长时间暂停）还是退出循环（在短暂或无暂停后）。而 `task_ratelimit` 的值，又最终由 `dirty` 和 `thresh` / `bg_thresh` 的相对关系决定。

