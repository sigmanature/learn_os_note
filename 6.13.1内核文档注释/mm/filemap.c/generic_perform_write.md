该函数是 VFS 的一部分，被许多文件系统（可能包括 F2FS 的缓冲路径）用于处理缓冲写入的核心循环。它与页缓存交互，并调用文件系统特定的钩子（`write_begin`、`write_end`）。

```c
// generic_perform_write: 用于缓冲写入的 VFS 辅助函数
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
	struct file *file = iocb->ki_filp; // 获取文件结构
	loff_t pos = iocb->ki_pos; // 当前写入位置（将被更新）
	struct address_space *mapping = file->f_mapping; // 获取地址空间（管理页缓存）
	const struct address_space_operations *a_ops = mapping->a_ops; // 获取文件系统特定的操作
	// 确定此映射的 folio（多页单元）的最大大小
	size_t chunk = mapping_max_folio_size(mapping);
	long status = 0; // 来自 a_ops 调用或错误的状态码
	ssize_t written = 0; // 成功写入的总字节数

	// 当迭代器 'i' 中还有数据时循环
	do {
		struct folio *folio; // 指向页缓存 folio 的指针
		size_t offset;		/* folio 内写入开始的偏移量 */
		size_t bytes;		/* 本次迭代中要写入此 folio 的字节数 */
		size_t copied;		/* 实际从用户空间复制的字节数 */
		void *fsdata = NULL; // 在 write_begin/write_end 之间传递的文件系统特定数据

		// 获取剩余要写入的总字节数
		bytes = iov_iter_count(i);
retry:
		// 计算 folio（页对齐块）内的偏移量
		offset = pos & (chunk - 1);
		// 计算本次迭代要写入多少字节：
		// folio 块中剩余字节数 或 总剩余字节数 中的较小值
		bytes = min(chunk - offset, bytes);
		// 如果系统脏页过多，则限制写入速率
		balance_dirty_pages_ratelimited(mapping);

		/*
		 * _首先_ 调入我们将从中复制的用户页。
		 * 否则，在从未标记为最新的同一页复制时，
		 * 会出现严重的死锁。
		 * （在锁定内核页缓存 folio 之前，先将用户缓冲区页面调入内存）
		 */
		if (unlikely(fault_in_iov_iter_readable(i, bytes) == bytes)) {
			status = -EFAULT; // 访问用户缓冲区错误
			break;
		}

		// 检查致命信号（例如 SIGKILL）
		if (fatal_signal_pending(current)) {
			status = -EINTR; // 被信号中断
			break;
		}

		/* --- 文件系统交互点 1: write_begin --- */
		// 调用文件系统的 write_begin 操作。
		// 目的：
		// 1. 查找或创建目标偏移量 'pos' 的页缓存 folio。
		// 2. 锁定 folio。
		// 3. 准备 folio 以供写入：
		//    - 如果是部分写入且 folio 不是最新的，则从磁盘读取现有数据。
		//    - **执行任何必要的块分配/预留（如 XFS 延迟分配或 F2FS 块预留）。**
		//    - 处理文件系统特定细节（内联数据、压缩准备等）。
		// 4. 通过 'foliop' 返回锁定的 folio 和可选的 fsdata。
		status = a_ops->write_begin(file, mapping, pos, bytes,
						&folio, &fsdata);
		if (unlikely(status < 0)) // 文件系统的 write_begin 失败
			break;

		// 在可能的多页 folio 中重新计算偏移量
		offset = offset_in_folio(folio, pos);
		// 确保 'bytes' 不超过从计算出的偏移量开始的 folio 边界
		if (bytes > folio_size(folio) - offset)
			bytes = folio_size(folio) - offset;

		// 如果页缓存直接映射到用户空间（罕见），刷新 CPU 缓存
		if (mapping_writably_mapped(mapping))
			flush_dcache_folio(folio);

		// 将数据从用户缓冲区 (iov_iter 'i') 复制到页缓存 folio
		// 这是关于页锁的原子复制。
		copied = copy_folio_from_iter_atomic(folio, offset, bytes, i);
		// 写入内核缓冲区后再次刷新 CPU 缓存
		flush_dcache_folio(folio);

		/* --- 文件系统交互点 2: write_end --- */
		// 调用文件系统的 write_end 操作。
		// 目的：
		// 1. 完成对 folio 的写入。
		// 2. 将 folio 的相关部分标记为脏页。
		// 3. 如有必要，更新文件系统元数据（例如，新分配块的块映射）。
		// 4. 解锁 folio。
		// 5. 如果使用了 fsdata，则进行清理。
		// 返回文件系统成功处理/写入的字节数。
		status = a_ops->write_end(file, mapping, pos, bytes, copied,
						folio, fsdata);
		if (unlikely(status != copied)) { // write_end 报告错误或写入字节数不足
			// 如果 write_end 失败或写入少于复制的字节数，则回滚迭代器
			iov_iter_revert(i, copied - max(status, 0L));
			if (unlikely(status < 0)) // 检查 write_end 是否返回错误
				break;
		}
		cond_resched(); // 如有需要，允许调度程序运行

		if (unlikely(status == 0)) { // 数据复制后 write_end 接受了 0 字节
			/*
			 * 一次短复制导致 ->write_end() 完全拒绝了
			 * 这次操作。可能是中途内存损坏，
			 * 可能是与 munmap 的竞争，
			 * 可能是严重的内存压力。
			 * 尝试使用更小的块大小重试。
			 */
			if (chunk > PAGE_SIZE) // 减小重试的块大小
				chunk /= 2;
			if (copied) { // 如果在拒绝前确实复制了一些东西
				bytes = copied; // 将 bytes 设置为已复制的数量以进行重试
				goto retry; // 重试已复制数量的操作
			}
			// 如果复制了 0 且状态为 0，则表示有问题，跳出循环。
		} else { // write_end 成功 (status > 0)
			// 推进位置和总写入计数
			pos += status;
			written += status;
		}
	} while (iov_iter_count(i)); // 如果迭代器中还有更多数据，则继续循环

	if (!written) // 如果根本没有写入任何内容，则返回状态/错误
		return status;
	iocb->ki_pos += written; // 更新 iocb 中的文件位置
	return written; // 返回总写入字节数
}
EXPORT_SYMBOL(generic_perform_write);
```

**`generic_perform_write` 的关键启示：**

*   **通用循环：** 这提供了标准的缓冲写入循环结构。
*   **文件系统钩子：** 文件系统特定行为的关键部分是 `a_ops->write_begin` 和 `a_ops->write_end`。
*   **`write_begin` 是关键：** 这是文件系统准备页缓存页面并在数据从用户复制*之前*处理块分配/预留逻辑的地方。这与 XFS iomap/bmapi 逻辑在缓冲写入期间运行的位置直接对应。