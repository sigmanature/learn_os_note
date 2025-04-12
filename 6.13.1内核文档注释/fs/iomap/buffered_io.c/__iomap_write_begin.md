**相关函数**
* [iomap_write_begin](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_write_begin.md)
**4. `__iomap_write_begin` (iomap folio 内容准备)**

此函数是 `iomap_write_begin` 的核心辅助函数，负责根据 iomap 的类型 (HOLE, DELALLOC, UNWRITTEN, MAPPED) 来准备 folio 的内容，确保在将用户数据复制进来之前，folio 中未被覆盖的部分包含正确的数据（零或从磁盘读取的数据）。

```c
// __iomap_write_begin - 准备 folio 内容 (处理清零或读取)
static int __iomap_write_begin(const struct iomap_iter *iter, loff_t pos,
		size_t len, struct folio *folio)
{
	const struct iomap *srcmap = iomap_iter_srcmap(iter); // 获取源映射
	struct iomap_folio_state *ifs; // 指向 folio 块级状态跟踪结构
	loff_t block_size = i_blocksize(iter->inode); // 获取 inode 的块大小
	// 计算写入范围涉及的起始块边界和结束块边界
	loff_t block_start = round_down(pos, block_size);
	loff_t block_end = round_up(pos + len, block_size);
	// 计算 folio 包含多少个 inode 块
	unsigned int nr_blocks = i_blocks_per_folio(iter->inode, folio);
	// 计算写入在 folio 内的起始偏移和结束偏移
	size_t from = offset_in_folio(folio, pos), to = from + len;
	size_t poff, plen; // 用于循环处理 folio 内块范围的变量

	/*
	 * 如果写入或清零完全覆盖了当前 folio，那么整个 folio
	 * 都将被标记为脏，因此无需将每个块的状态跟踪结构附加到此 folio。
	 * 对于 unshare 情况，我们必须读入磁盘内容，因为我们不改变页缓存内容。
	 */
	// 优化：如果写入覆盖整个 folio (并且不是 unshare 操作)，则无需进一步处理，
	// 因为用户数据将覆盖所有内容。
	if (!(iter->flags & IOMAP_UNSHARE) && pos <= folio_pos(folio) &&
	    pos + len >= folio_pos(folio) + folio_size(folio))
		return 0; // 直接返回成功

	// --- Folio 状态跟踪 ---
	// 为 folio 分配或获取块级状态跟踪结构 (iomap_folio_state)
	// 这个结构用于跟踪 folio 内每个块的 uptodate 状态，特别是在部分写入或读取时。
	ifs = ifs_alloc(iter->inode, folio, iter->flags);
	// 如果是 NOWAIT 模式且无法立即分配 ifs (且 folio 包含多个块)，则返回 EAGAIN
	if ((iter->flags & IOMAP_NOWAIT) && !ifs && nr_blocks > 1)
		return -EAGAIN;

	// 如果 folio 已经是 up-to-date (内容有效)，则无需读取或清零
	if (folio_test_uptodate(folio))
		return 0;

	// --- 处理非 up-to-date 的 folio ---
	// 循环遍历写入范围 [pos, pos+len) 所涉及的 inode 块
	do {
		// 调整读取/处理范围，使其与 folio 内的块对齐，并获取块在 folio 内的偏移 (poff) 和长度 (plen)
		iomap_adjust_read_range(iter->inode, folio, &block_start,
				block_end - block_start, &poff, &plen);
		if (plen == 0) // 如果没有更多块需要处理，则退出循环
			break;

		// 检查当前块 [poff, poff+plen) 是否与写入范围 [from, to) 有重叠
		// 如果没有重叠 (并且不是 unshare 操作)，则跳过此块
		if (!(iter->flags & IOMAP_UNSHARE) &&
		    (from <= poff || from >= poff + plen) &&
		    (to <= poff || to >= poff + plen))
			continue; // 跳到下一个块

		// --- 判断是清零还是读取 ---
		// iomap_block_needs_zeroing: 检查当前块是否需要清零。
		// 这通常基于 iomap 类型：HOLE, DELALLOC, UNWRITTEN 需要清零。
		// MAPPED 类型通常不需要清零 (需要读取)。
		if (iomap_block_needs_zeroing(iter, block_start)) {
			// 如果是 unshare 操作却需要清零，这是个错误
			if (WARN_ON_ONCE(iter->flags & IOMAP_UNSHARE))
				return -EIO;
			// 清零 folio 中与当前块对应的、且未被写入范围 [from, to) 覆盖的部分
			folio_zero_segments(folio, poff, from, to, poff + plen);
		} else { // 块不需要清零，意味着需要从磁盘读取 (通常是 MAPPED 类型)
			int status;

			// 如果是 NOWAIT 模式，不能阻塞读取，返回 EAGAIN
			if (iter->flags & IOMAP_NOWAIT)
				return -EAGAIN;

			// 同步读取块内容到 folio 的相应位置
			status = iomap_read_folio_sync(block_start, folio,
					poff, plen, srcmap);
			if (status) // 读取失败
				return status;
		}
		// 标记 folio 中当前处理的块范围 [poff, poff+plen) 为 up-to-date
		iomap_set_range_uptodate(folio, poff, plen);
	// 继续处理下一个块，更新 block_start
	} while ((block_start += plen) < block_end);

	// 所有涉及的块都已处理 (清零或读取)
	return 0; // 成功
}
```