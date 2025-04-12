
好的，让我们深入研究 F2FS 缓冲写入路径，从 VFS 层开始，一直到 F2FS 的 `write_begin`。我们将在进行过程中与 XFS 进行比较和对比。

**1. `f2fs_file_write_iter` (VFS `writev`/`pwritev` 的入口点)**

当应用程序在 F2FS 文件上调用 `writev`、`pwritev` 等时，此函数是主要的入口点。它负责协调整个写入操作。

```c
// f2fs_file_write_iter: F2FS 对写操作 (writev, pwritev) 的实现
static ssize_t f2fs_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
	struct inode *inode = file_inode(iocb->ki_filp); // 获取与文件关联的 inode
	const loff_t orig_pos = iocb->ki_pos; // 保存原始位置用于追踪/同步
	const size_t orig_count = iov_iter_count(from); // 保存原始计数用于追踪/同步
	loff_t target_size; // 这次写入后的预期最终大小
	bool dio; // 标志：直接 I/O 还是缓冲 I/O？
	bool may_need_sync = true; // 标志：这次写入后续是否可能需要 fsync？（对于缓冲写入通常为 true）
	int preallocated; // 预分配尝试的结果
	// 注意：pos 和 count 不是 const，iocb->ki_pos 将由底层的写函数更新
	const loff_t pos = iocb->ki_pos;
	const ssize_t count = iov_iter_count(from);
	ssize_t ret; // 返回值（写入的字节数或错误码）

	// 检查文件系统在检查点期间是否遇到严重错误
	if (unlikely(f2fs_cp_error(F2FS_I_SB(inode)))) {
		ret = -EIO; // 文件系统处于只读错误状态
		goto out;
	}

	// 检查压缩后端是否准备就绪（如果启用了压缩）
	if (!f2fs_is_compress_backend_ready(inode)) {
		ret = -EOPNOTSUPP; // 请求了压缩但不可用/未就绪
		goto out;
	}

	// 获取 inode 锁（标准的 VFS 锁）
	// IOCB_NOWAIT 允许非阻塞尝试
	if (iocb->ki_flags & IOCB_NOWAIT) {
		if (!inode_trylock(inode)) { // 尝试非阻塞地获取锁
			ret = -EAGAIN; // 无法立即获取锁
			goto out;
		}
	} else {
		inode_lock(inode); // 等待 inode 锁
	}

	// 检查固定文件（类似 immutable 属性的概念）
	// 通常不允许覆盖固定文件，除非是完全覆盖？(f2fs_overwrite_io 检查)
	if (f2fs_is_pinned_file(inode) &&
	    !f2fs_overwrite_io(inode, pos, count)) {
		ret = -EIO; // 尝试不适当地修改固定文件
		goto out_unlock;
	}

	// 执行通用的写入检查（权限、限制、强制锁定等）
	ret = f2fs_write_checks(iocb, from);
	if (ret <= 0) // 错误或请求写入 0 字节
		goto out_unlock;

	/* 确定我们将执行直接写入还是缓冲写入。*/
	// 根据 inode 标志、文件标志 (O_DIRECT)、对齐等检查是否应使用直接 I/O
	dio = f2fs_should_use_dio(inode, iocb, from);

	/* dio 与原子写入不兼容 */
	// F2FS 原子写入依赖于 CoW 和页缓存机制，与绕过缓存的 DIO 不兼容。
	if (dio && f2fs_is_atomic_file(inode)) {
		ret = -EOPNOTSUPP; // 不能对原子文件执行 DIO
		goto out_unlock;
	}

	/* 可能为写入预分配块。*/
	// 计算潜在的新文件结束位置
	target_size = iocb->ki_pos + iov_iter_count(from);
	// 调用 f2fs_preallocate_blocks。这是在主写入循环之前的 F2FS 特定步骤。
	// 它尝试预先保留块，对于 DIO 或扩展写入特别有用。
	// 这与 XFS 的延迟分配不同，后者发生在缓冲写入的 write_begin 期间。
	preallocated = f2fs_preallocate_blocks(iocb, from, dio);
	if (preallocated < 0) { // 预分配失败（例如，ENOSPC）
		ret = preallocated;
	} else { // 预分配成功或不需要/未尝试
		// 如果配置了追踪，则启用
		if (trace_f2fs_datawrite_start_enabled())
			f2fs_trace_rw_file_path(iocb->ki_filp, iocb->ki_pos,
						orig_count, WRITE);

		/* 执行实际的写入。*/
		// 根据 'dio' 标志分派到相应的写入函数
		ret = dio ?
			f2fs_dio_write_iter(iocb, from, &may_need_sync) : // 直接 I/O 路径
			f2fs_buffered_write_iter(iocb, from); // 缓冲 I/O 路径（可能调用 generic_perform_write）

		// 追踪写入结束
		if (trace_f2fs_datawrite_end_enabled())
			trace_f2fs_datawrite_end(inode, orig_pos, ret);
	}

	/* 不要让任何预分配的块遗留在 i_size 之后。*/
	// 如果预分配了块 (`preallocated > 0`) 并且写入实际上没有将文件大小扩展到
	// 目标大小（例如，写入不完整、错误），我们需要截断未使用的预分配空间。
	if (preallocated && i_size_read(inode) < target_size) {
		// 获取 F2FS 特定的用于垃圾回收路径的写锁
		f2fs_down_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
		// 锁定页缓存以进行失效/截断
		filemap_invalidate_lock(inode->i_mapping);
		// 将 inode 截断回其当前的有效大小
		if (!f2fs_truncate(inode)) // f2fs_truncate 成功时返回 0
			file_dont_truncate(inode); // 标记 inode 以防止 VFS 层进行冗余截断
		filemap_invalidate_unlock(inode->i_mapping);
		f2fs_up_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
	} else {
		// 标记 inode 以防止 VFS 层进行冗余截断
		file_dont_truncate(inode);
	}

	// 清除指示所有块都已预分配的标志（在 prepare_write_begin 优化中使用）
	clear_inode_flag(inode, FI_PREALLOCATED_ALL);
out_unlock:
	inode_unlock(inode); // 释放 inode 锁
out:
	// 追踪 write_iter 操作的结束
	trace_f2fs_file_write_iter(inode, orig_pos, orig_count, ret);

	// 如果写入成功 (ret > 0) 并且是缓冲写入 (may_need_sync 为 true)，
	// 调用 generic_write_sync 处理 O_SYNC/O_DSYNC 标志。
	if (ret > 0 && may_need_sync)
		ret = generic_write_sync(iocb, ret);

	/* 如果强制使用了缓冲 IO，则刷新并从页缓存中丢弃数据
	 * 以保持 O_DIRECT 语义
	 */
	// 这处理了请求 O_DIRECT 但不可能（例如，由于对齐）的情况，
	// 因此使用了缓冲 I/O。为了模拟 O_DIRECT，刷新写入的数据并使缓存失效。
	if (ret > 0 && !dio && (iocb->ki_flags & IOCB_DIRECT))
		f2fs_flush_buffered_write(iocb->ki_filp->f_mapping,
					  orig_pos,
					  orig_pos + ret - 1);

	return ret; // 返回写入的字节数或错误码
}
```

**`f2fs_file_write_iter` 与 XFS 的关键区别：**

*   **预分配步骤：** F2FS 在主写入循环*之前*有一个显式的 `f2fs_preallocate_blocks` 调用。这似乎旨在预先保留空间，可能改善后续写入的布局或满足 DIO 要求。XFS 的主要预分配（EOF 扩展）和延迟分配发生在缓冲写入路径*内部*（`xfs_buffered_write_iomap_begin` / `xfs_bmapi_reserve_delalloc`）。
*   **分派：** 两个文件系统最终都会分派到 DIO 或缓冲写入路径。F2FS 使用 `f2fs_dio_write_iter` / `f2fs_buffered_write_iter`。XFS 使用 iomap，它内部处理 DIO 或回调到缓冲写入逻辑。
*   **缓冲路径：** F2FS 的 `f2fs_buffered_write_iter` 很可能调用 `generic_perform_write`（接下来展示），后者依赖于 `address_space_operations`。

**2. `generic_perform_write` (通用 VFS 缓冲写入辅助函数)**

* [generic_perform_write](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/generic_perform_write.md)

**3. `f2fs_write_begin` (F2FS `a_ops->write_begin` 实现)**

* [f2fs_write_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_begin.md)

**4. `prepare_atomic_write_begin` (原子写入辅助函数)**

[prepare_atomic_write_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/prepare_atomic_write_begin.md)

**5. `prepare_write_begin` (普通写入辅助函数)**

[prepare_write_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/prepare_write_begin.md)

**F2FS `write_begin` vs XFS `iomap_begin` / `bmapi`:**

*   **延迟分配:**
    *   **XFS:** 在 `write_begin` (`xfs_bmapi_reserve_delalloc`) 期间显式地在内存中创建“延迟分配”区段 (`nullstartblock`)。实际的物理块分配被推迟到回写阶段。
    *   **F2FS:** 对于普通写入，似乎没有直接等效的持久“延迟分配”状态。`prepare_write_begin` 要么找到现有的块地址，要么确定需要一个新块 (`NEW_ADDR`)。预留 (`f2fs_reserve_block`) 会更新元数据指针/空间计数器，但 `NEW_ADDR` 的物理块号的实际分配似乎与 F2FS 的 LFS（日志结构文件系统）特性相关，并在段写入/检查点期间进行，不一定像 XFS 的 delalloc 那样精确地延迟。`FI_PREALLOCATED_ALL` 优化最接近，它推迟了查找，但依赖于之前的 `f2fs_preallocate_blocks`。
*   **未写入区段:**
    *   **XFS:** 对于已分配但尚未写入有效数据的块（例如，通过 `fallocate`），有一个显式的 `XFS_EXT_UNWRITTEN` 状态。`iomap` 使用此状态在读取时返回零，而无需磁盘 I/O。
    *   **F2FS:** 在其块映射中似乎没有以相同方式跟踪的明确“未写入”状态。如果 `f2fs_write_begin` 找到一个现有块（`blk_addr != NULL_ADDR` 且 `!= NEW_ADDR`）但页缓存 folio 不是最新的，它会**发起读取** (`f2fs_submit_page_read`)。如果它确定块是新的 (`blk_addr == NEW_ADDR`)，它会**将 folio 清零** (`folio_zero_segment`)。F2FS 通过页缓存清零来处理空洞/新块的“读取返回零”语义，并通过先读取来处理已分配但过时的块。对 `fallocate` 的支持可能使用 `f2fs_reserve_block` 来分配块，并可能在不写入的情况下将页面标记为最新，但其*持久*状态似乎不如 XFS 的 `UNWRITTEN` 那样明确。
*   **原子写入/CoW:** F2FS 具有用于原子写入的内置 CoW 逻辑 (`prepare_atomic_write_begin`)，涉及一个单独的 `cow_inode` 和特定的预留 (`__reserve_data_block`)。这是 XFS 以不同方式处理的功能（例如，通过 reflink/快照，通常不是以相同方式处理每个文件的原子写入）。
*   **内联数据/压缩:** F2FS 在 `write_begin` 路径中直接对这些特性进行了特定处理，增加了基本 XFS 路径中不存在的复杂性。
*   **预分配:** F2FS 的前期 `f2fs_preallocate_blocks` 与 XFS 在写入期间集成的 EOF 预分配相比，在策略上存在显著差异。

本质上，虽然两者都旨在准备一个页缓存 folio 以供写入，但它们处理块分配和现有数据的策略差异很大，这受到 F2FS 的 LFS 设计和原子写入等特性的影响，而 XFS 则基于区段的分配和显式的延迟/未写入状态。F2FS 似乎在 `write_begin` 中更直接地解析块状态（存在/空洞/新），依赖于读取或清零，而 XFS 使用特定的区段状态（延迟、未写入）来推迟决策或优化读取。

