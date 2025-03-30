**1. `writeback_control` 结构体**

`writeback_control` 它是一个控制结构，用于告知 writeback 代码应该做什么。这些结构体通常在栈上分配，因此不需要显式的锁机制。它们在初始化时，未明确赋值的成员会被设置为零。

```c
struct writeback_control {
	/* 公共字段，可以被调用者设置和/或使用 */
	long nr_to_write;		/* 要写入的页数，每写入一页就递减 */
	long pages_skipped;		/* 被跳过的页数 (未写入的页数) */

	/*
	 * 对于 a_ops->writepages(): 如果 start 或 end 非零，则这是一个提示，
	 * 文件系统只需要写出该字节范围内 (start 到 end，包含 end) 的页。
	 */
	loff_t range_start;
	loff_t range_end;

	enum writeback_sync_modes sync_mode; // 同步模式

	unsigned for_kupdate:1;		/* kupdate writeback (内核定时器触发的 writeback) */
	unsigned for_background:1;	/* 后台 writeback */
	unsigned tagged_writepages:1;	/* 标记和写入页，避免活锁 */
	unsigned for_reclaim:1;		/* 页回收 (page reclaim) 触发的 writeback */
	unsigned range_cyclic:1;	/* range_start 是循环的 (用于某些特殊范围) */
	unsigned for_sync:1;		/* sync(2) 系统调用触发的 WB_SYNC_ALL writeback */
	unsigned unpinned_netfs_wb:1;	/* 清除 I_PINNING_NETFS_WB 标志 (用于网络文件系统) */

	/*
	 * 当 writeback I/O 通过异步层 bounce 时，只有初始的同步阶段
	 * 应该计入 inode cgroup 所有权仲裁，以避免混淆。
	 * 后续阶段可以设置以下标志来禁用记账。
	 */
	unsigned no_cgroup_owner:1;

	/* 为了启用 swap 写入到非块设备后端的批处理，
	 * "plug" 可以设置为指向 'struct swap_iocb *'。 当所有 swap
	 * 写入都已提交时，如果 swap_iocb 不为 NULL，则应调用
	 * swap_write_unplug()。
	 */
	struct swap_iocb **swap_plug;

	/* 用于拆分大型 folio 的目标列表 */
	struct list_head *list;

	/* ->writepages 实现使用的内部字段: */
	struct folio_batch fbatch; // Folio 批处理结构
	pgoff_t index; // 当前处理的页索引
	int saved_err; // 保存的错误码

#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback *wb;	/* 此 writeback 操作所属的 bdi_writeback 结构 */
	struct inode *inode;		/* 正在被写出的 inode */

	/* 外来 inode 检测，参见 wbc_detach_inode() */
	int wb_id;			/* 当前 wb id */
	int wb_lcand_id;		/* 上一个外来候选 wb id */
	int wb_tcand_id;		/* 当前外来候选 wb id */
	size_t wb_bytes;		/* 当前 wb 写入的字节数 */
	size_t wb_lcand_bytes;		/* 上一个候选写入的字节数 */
	size_t wb_tcand_bytes;		/* 当前候选写入的字节数 */
#endif
};
```

**`writeback_control` 结构体成员详细解析:**

* **公共字段 (Caller-Settable/Usable Fields):**

    * **`long nr_to_write;`**:  **要写入的页数限制。**  调用者可以设置这个字段来限制本次 writeback 操作最多写入多少页。  Writeback 代码在每次成功写入一页后会递减这个计数器。当计数器降至 0 或更小时，writeback 操作可能会停止。  设置为 `LONG_MAX` 通常表示没有页数限制，尽可能多地写回脏页。

    * **`long pages_skipped;`**:  **被跳过的页数。**  用于记录在 writeback 过程中，由于各种原因 (例如，页不是脏的、页被锁定、达到跳过条件等) 而没有被写入的页数。

    * **`loff_t range_start;`**:  **写回范围的起始字节偏移量。**  用于指定只写回文件指定字节范围内的脏页。  通常与 `range_end` 配合使用。

    * **`loff_t range_end;`**:  **写回范围的结束字节偏移量 (包含)。**  与 `range_start` 一起定义了要写回的文件字节范围。  设置为 `LLONG_MAX` 通常表示写到文件末尾。

    * **`enum writeback_sync_modes sync_mode;`**:  **同步模式。**  定义了 writeback 操作的同步级别，决定了 writeback 操作是否需要同步等待完成。  可能的取值 (定义在 `fs/fs-writeback.c` 的 `enum writeback_sync_modes`):
        * `WB_SYNC_NONE`:  **异步 writeback。**  启动 writeback 操作后立即返回，不等待 I/O 完成。  通常用于后台 writeback 或内存回收。
        * `WB_SYNC_ALL`:  **同步 writeback。**  启动 writeback 操作后，需要等待所有相关的 I/O 操作完成才能返回。  通常用于 `fsync` 系统调用等需要数据持久性的场景。

    * **`unsigned for_kupdate:1;`**:  **kupdate writeback 标志。**  如果设置为 1，表示这次 writeback 是由内核定时器 `kupdate` 触发的。  `kupdate` 定期唤醒 writeback 守护进程，进行周期性的脏页写回。

    * **`unsigned for_background:1;`**:  **后台 writeback 标志。**  如果设置为 1，表示这次 writeback 是后台 writeback 进程 (如 `pdflush` 或 `bdi-default`) 触发的。  后台 writeback 通常用于在系统空闲时清理脏页，减少内存压力。

    * **`unsigned tagged_writepages:1;`**:  **标记和写入页标志。**  用于标记是否使用 "tagged writepages" 机制，以避免在某些情况下 (例如，高负载、内存压力) 出现活锁 (livelock)。  Tagged writepages 是一种更精细的控制 writeback 顺序和优先级的机制。

    * **`unsigned for_reclaim:1;`**:  **页回收 writeback 标志。**  如果设置为 1，表示这次 writeback 是由内存回收 (page reclaim) 机制触发的。  当系统内存不足时，内核会触发页回收，将一些脏页写回磁盘以释放内存。

    * **`unsigned range_cyclic:1;`**:  **循环范围标志。**  用于指示 `range_start` 是否是循环的。  在某些特殊情况下，例如处理回环设备或某些特殊文件系统时，可能需要使用循环范围。

    * **`unsigned for_sync:1;`**:  **`sync(2)` 系统调用 writeback 标志。**  如果设置为 1，表示这次 writeback 是由 `sync(2)` 系统调用 (或相关系统调用，如 `fsync`, `fdatasync`) 触发的，并且同步模式是 `WB_SYNC_ALL`。

    * **`unsigned unpinned_netfs_wb:1;`**:  **网络文件系统 unpinned writeback 标志。**  用于网络文件系统 (如 NFS, SMB/CIFS)。  当网络文件系统的页被 unpinned (解除网络文件系统特定的 pinning) 时，可以设置此标志来触发 writeback。  `I_PINNING_NETFS_WB` 标志可能与网络文件系统的缓存一致性有关。

    * **`unsigned no_cgroup_owner:1;`**:  **禁用 cgroup 所有者记账标志。**  在 writeback I/O 通过异步层 bounce 时，只有初始的同步阶段应该计入 inode cgroup 所有权仲裁，以避免混淆。  后续阶段可以设置此标志来禁用 cgroup 记账。  这与 cgroup writeback 功能有关，用于资源控制和隔离。

    * **`struct swap_iocb **swap_plug;`**:  **swap 写入批处理 plug。**  用于 swap 写入到非块设备后端 (例如，zswap, 内存压缩交换)。  可以设置为指向 `struct swap_iocb *`，用于批处理 swap 写入操作。  当所有 swap 写入都提交后，如果 `swap_plug` 不为 NULL，则应调用 `swap_write_unplug()` 来完成批处理。

    * **`struct list_head *list;`**:  **folio 拆分目标列表。**  用于在处理大型 folio 时，将 folio 拆分成更小的部分，并将这些部分添加到 `list` 指向的链表中。  可能用于优化大型 folio 的处理，例如，在内存压力下或为了提高并发性。

* **内部字段 (Internal Fields):**

    * **`struct folio_batch fbatch;`**:  **folio 批处理结构。**  用于在 `writepages` 操作中，批量处理 folio，提高效率。  `folio_batch` 结构体可能用于缓存 folio 指针，减少重复查找和锁竞争。

    * **`pgoff_t index;`**:  **当前处理的页索引。**  在 `writepages` 迭代页缓存时，`index` 记录当前正在处理的页的索引 (页偏移量)。

    * **`int saved_err;`**:  **保存的错误码。**  用于在 writeback 过程中保存遇到的错误码。  例如，在批量写回多个页时，如果其中一个页的写回操作失败，可以将错误码保存在 `saved_err` 中，并在最后返回。

    * ### `#ifdef CONFIG_CGROUP_WRITEBACK`部分
    * **`struct bdi_writeback *wb;`**:  **所属的 `bdi_writeback` 结构体指针。**  指向当前 writeback 操作所属的 `bdi_writeback` 结构体。  `bdi_writeback` 结构体负责管理和控制特定 BDI (Backing Device Info) 的 writeback 活动。

    * **`struct inode *inode;`**:  **正在被写出的 inode 指针。**  指向当前正在进行 writeback 操作的 inode。

    * **外来 inode 检测相关字段 (`wb_id`, `wb_lcand_id`, `wb_tcand_id`, `wb_bytes`, `wb_lcand_bytes`, `wb_tcand_bytes`):**  这些字段用于 **外来 inode 检测 (foreign inode detection)**。  外来 inode 指的是不属于当前 cgroup 的 inode。  在 cgroup writeback 场景下，需要检测外来 inode，并可能进行特殊处理，以避免资源争用和性能问题。  这些字段用于跟踪 writeback 操作的 ID、候选 ID、写入字节数等信息，用于判断是否是外来 inode 导致的 writeback 延迟或性能下降。  具体细节可以参考 `wbc_detach_inode()` 函数和 cgroup writeback 相关文档。

**总结 `writeback_control` 结构体:**

`writeback_control` 结构体是一个功能非常强大的控制中心，它 **封装了 writeback 操作的所有关键参数和状态信息**。  通过设置 `writeback_control` 的不同字段，可以灵活地控制 writeback 的行为，例如同步/异步模式、写回范围、页数限制、触发原因等。  理解 `writeback_control` 结构体对于深入理解 Linux 内核 writeback 机制至关重要。
