 ```c
// iomap_write_begin - 获取并准备单个 folio 以供写入
static int iomap_write_begin(struct iomap_iter *iter, loff_t pos,
		size_t len, struct folio **foliop)
{
	// 获取文件系统提供的 folio 操作回调 (如果存在)
	const struct iomap_folio_ops *folio_ops = iter->iomap.folio_ops;
	// 获取源映射信息 (通常与 iter->iomap 相同，除非有特殊情况如 CoW)
	const struct iomap *srcmap = iomap_iter_srcmap(iter);
	struct folio *folio; // folio 指针
	int status = 0; // 状态码

	// 检查写入范围是否超出当前 iomap 映射范围 (逻辑错误)
	BUG_ON(pos + len > iter->iomap.offset + iter->iomap.length);
	// 如果使用了源映射 (srcmap)，也检查范围
	if (srcmap != &iter->iomap)
		BUG_ON(pos + len > srcmap->offset + srcmap->length);

	// 检查是否有待处理的致命信号
	if (fatal_signal_pending(current))
		return -EINTR;

	// 如果 address_space 不支持大 folio，则将长度限制在单个页面内
	if (!mapping_large_folio_support(iter->inode->i_mapping))
		len = min_t(size_t, len, PAGE_SIZE - offset_in_page(pos));

	// --- 获取/创建并锁定 Folio ---
	// 调用 iomap 内部函数获取 folio，处理创建、查找和锁定
	folio = __iomap_get_folio(iter, pos, len);
	if (IS_ERR(folio)) // 获取 folio 失败
		return PTR_ERR(folio);

	/*
	 * 现在我们有了一个锁定的 folio，在对其进行任何操作之前，我们需要
	 * 检查我们缓存的 iomap 是否过时。inode 区段映射可能会因
	 * 并发的 IO 而改变（例如，IOMAP_UNWRITTEN 状态可能改变，
	 * 内存回收可能在 IO 完成后、此写入到达此文件偏移量之前，
	 * 回收了先前部分写入的此索引处的页面），因此我们
	 * 可能在这里做出错误的操作（错误地清零页面范围或未能清零）
	 * 并损坏数据。
	 */
	// 如果文件系统提供了 iomap_valid 回调
	if (folio_ops && folio_ops->iomap_valid) {
		// 调用回调函数检查当前 iter->iomap 是否仍然有效
		bool iomap_valid = folio_ops->iomap_valid(iter->inode,
							 &iter->iomap);
		if (!iomap_valid) { // 如果 iomap 无效
			iter->iomap.flags |= IOMAP_F_STALE; // 标记 iomap 已失效
			status = 0; // 返回 0，但设置了 STALE 标志
			goto out_unlock; // 跳转到解锁并返回
		}
	}

	// 确保处理的长度不超过 folio 的实际边界
	if (pos + len > folio_pos(folio) + folio_size(folio))
		len = folio_pos(folio) + folio_size(folio) - pos;

	// --- 根据 iomap 类型分派处理 ---
	if (srcmap->type == IOMAP_INLINE) // 如果是内联数据类型
		// 调用内联数据的特定处理函数 (可能需要转换)
		status = iomap_write_begin_inline(iter, folio);
	else if (srcmap->flags & IOMAP_F_BUFFER_HEAD) // 如果使用旧的 buffer_head 方式
		// 调用基于 buffer_head 的块写入准备函数
		status = __block_write_begin_int(folio, pos, len, NULL, srcmap);
	else // 其他类型 (HOLE, DELALLOC, UNWRITTEN, MAPPED)
		// 调用通用的 iomap 写入准备函数
		status = __iomap_write_begin(iter, pos, len, folio);

	if (unlikely(status)) // 如果准备失败
		goto out_unlock; // 跳转到解锁并返回

	// 成功，将获取到的 folio 返回给调用者
	*foliop = folio;
	return 0;

out_unlock: // 错误处理或 iomap 失效路径
	// 释放 folio (解锁并减少引用计数)
	__iomap_put_folio(iter, pos, 0, folio); // 写入 0 字节

	return status; // 返回错误码或 0 (如果 iomap 失效)
}
```