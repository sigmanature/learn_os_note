**相关函数**
* [iomap_write_begin](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_write_begin.md)
**2. `iomap_write_iter` (iomap 缓冲写入核心循环)**

这个函数处理 `iomap_iter` 返回的**单个**映射区域 (`iter.iomap`) 内的缓冲写入。它的结构类似于 `generic_perform_write`，但它是 iomap 框架的一部分，并调用 iomap 内部的 `iomap_write_begin/end`。

```c
// iomap_write_iter - 处理单个 iomap 映射区域的缓冲写入
static loff_t iomap_write_iter(struct iomap_iter *iter, struct iov_iter *i)
{
	// 获取当前 iomap 映射区域的长度
	loff_t length = iomap_length(iter);
	// 当前处理的位置 (在 iter->iomap 区域内)
	loff_t pos = iter->pos;
	ssize_t total_written = 0; // 在此 iomap 区域内总共写入的字节数
	long status = 0; // 状态码
	struct address_space *mapping = iter->inode->i_mapping; // 获取地址空间
	// 获取映射支持的最大 folio 大小
	size_t chunk = mapping_max_folio_size(mapping);
	// 脏页平衡标志，如果是 NOWAIT，则使用异步模式
	unsigned int bdp_flags = (iter->flags & IOMAP_NOWAIT) ? BDP_ASYNC : 0;

	// 循环处理当前 iomap 映射区域 (length) 或者直到用户数据 (iov_iter i) 处理完毕
	do {
		struct folio *folio; // 页缓存 folio
		loff_t old_size;    // 写入前的 inode 大小
		size_t offset;		/* folio 内的偏移量 */
		size_t bytes;		/* 本次迭代要写入 folio 的字节数 */
		size_t copied;		/* 从用户空间复制的字节数 */
		size_t written;		/* 实际写入的字节数 (由 iomap_write_end 确认) */

		// 获取剩余要写入的总字节数 (来自用户)
		bytes = iov_iter_count(i);
retry:
		// 计算在 folio (块大小为 chunk) 内的偏移量
		offset = pos & (chunk - 1);
		// 计算本次迭代要写入的字节数：folio 内剩余空间、用户剩余数据、当前 iomap 区域剩余长度中的最小值
		bytes = min(chunk - offset, bytes);
		// 进行脏页回写控制 (限速)
		status = balance_dirty_pages_ratelimited_flags(mapping,
							       bdp_flags);
		if (unlikely(status)) // 如果需要等待回写且出错/中断
			break;

		// 确保写入不超过当前 iomap 映射区域的剩余长度
		if (bytes > length)
			bytes = length;

		/*
		 * _首先_ 调入我们将从中复制的用户页。
		 * 否则，在从未标记为最新的同一页复制时，
		 * 会出现严重的死锁。
		 *
		 * 对于异步缓冲写入，假设用户页已经被调入。
		 * 可以通过调入用户页来优化。
		 */
		if (unlikely(fault_in_iov_iter_readable(i, bytes) == bytes)) {
			status = -EFAULT; // 访问用户缓冲区错误
			break;
		}

		// --- iomap 框架的 folio 准备 ---
		// 调用 iomap 内部的 write_begin 函数
		// 目的：获取/创建/锁定 folio，并根据 iter->iomap 的信息
		// (类型: HOLE, DELALLOC, UNWRITTEN, MAPPED) 准备 folio 内容
		// (可能需要清零或读取旧数据)。
		status = iomap_write_begin(iter, pos, bytes, &folio);
		if (unlikely(status)) { // iomap_write_begin 失败
			iomap_write_failed(iter->inode, pos, bytes); // 通知文件系统写入失败 (可选回调)
			break;
		}
		// 检查 iomap_write_begin 是否发现 iomap 映射已失效
		if (iter->iomap.flags & IOMAP_F_STALE)
			break; // 如果失效，则退出此内层循环，外层 iomap_iter 会重新获取映射

		// 重新计算 folio 内的偏移量 (folio 可能比请求的 bytes 大)
		offset = offset_in_folio(folio, pos);
		// 确保写入不超过 folio 的边界
		if (bytes > folio_size(folio) - offset)
			bytes = folio_size(folio) - offset;

		// 如果页缓存被直接映射，刷新 CPU dcache
		if (mapping_writably_mapped(mapping))
			flush_dcache_folio(folio);

		// 从用户缓冲区 (i) 原子地复制数据到 folio
		copied = copy_folio_from_iter_atomic(folio, offset, bytes, i);
		// --- iomap 框架的 folio 完成 ---
		// 调用 iomap 内部的 write_end 函数
		// 目的：标记 folio 脏，处理元数据更新（如果需要），最终会解锁 folio (在 __iomap_put_folio 中)
		// 返回值表示是否成功处理了 copied 字节
		written = iomap_write_end(iter, pos, bytes, copied, folio) ?
			  copied : 0;

		/*
		 * 在将数据复制到页缓存后更新内存中的 inode 大小。
		 * 文件系统负责将更新后的大小写入磁盘，最好是在 I/O 完成后，
		 * 以免暴露陈旧数据。只有在那之后，我们才能解锁并释放 folio。
		 * (注意：解锁释放在 __iomap_put_folio 中完成)
		 */
		old_size = iter->inode->i_size; // 获取旧的 inode 大小
		if (pos + written > old_size) { // 如果写入扩展了文件
			i_size_write(iter->inode, pos + written); // 更新内存中的 inode 大小
			iter->iomap.flags |= IOMAP_F_SIZE_CHANGED; // 标记大小已改变
		}
		// 释放 folio (处理引用计数、解锁等)
		__iomap_put_folio(iter, pos, written, folio);

		// 如果写入造成了文件空洞 (old_size < pos)，通知页缓存
		if (old_size < pos)
			pagecache_isize_extended(iter->inode, old_size, pos);

		cond_resched(); // 允许调度
		if (unlikely(written == 0)) { // iomap_write_end 拒绝了写入 (返回 0)
			/*
			 * 一次短复制导致 iomap_write_end() 完全拒绝了
			 * 这次操作。可能是中途内存损坏，可能是与 munmap 的竞争，
			 * 可能是严重的内存压力。
			 */
			iomap_write_failed(iter->inode, pos, bytes); // 通知文件系统写入失败
			iov_iter_revert(i, copied); // 回滚用户迭代器

			if (chunk > PAGE_SIZE) // 减小块大小重试
				chunk /= 2;
			if (copied) { // 如果确实复制了数据
				bytes = copied;
				goto retry; // 重试
			}
			// 如果没复制数据也被拒绝，则退出
		} else { // 写入成功
			pos += written;        // 更新当前 iomap 区域内的位置
			total_written += written; // 累加在此 iomap 区域写入的字节数
			length -= written;      // 减少当前 iomap 区域的剩余长度
		}
	// 继续循环，直到用户数据写完 (iov_iter_count(i) == 0) 或当前 iomap 区域处理完 (length == 0)
	} while (iov_iter_count(i) && length);

	// 如果是因为需要等待而退出 (例如 NOWAIT 模式下 balance_dirty_pages 返回 -EAGAIN)
	if (status == -EAGAIN) {
		iov_iter_revert(i, total_written); // 回滚已处理的部分
		return -EAGAIN; // 返回 EAGAIN
	}
	// 返回在此 iomap 区域内成功写入的字节数，或者返回错误码 status
	return total_written ? total_written : status;
}
```

