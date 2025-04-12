这是 XFS 为缓冲写入定制的 `iomap_begin` 回调函数，是整个缓冲写入逻辑中最复杂的部分。它负责查找或创建文件在指定偏移量处的块映射，处理延迟分配、CoW、预分配等。

```c
static int // 函数成功返回 0 (并在 iomap 中填充映射信息)，失败返回负错误码
xfs_buffered_write_iomap_begin(
	struct inode		*inode,   // VFS inode
	loff_t			offset,   // 写入的起始偏移量 (字节)
	loff_t			count,    // 请求写入的长度 (字节)
	unsigned		flags,    // iomap 操作标志 (如 IOMAP_WRITE, IOMAP_ZERO, IOMAP_UNSHARE)
	struct iomap		*iomap,   // [输出] 存储找到或创建的映射信息
	struct iomap		*srcmap)  // [输出] 存储源映射信息 (主要用于 CoW)
{
	// 获取 XFS inode 和 mount point
	struct xfs_inode	*ip = XFS_I(inode);
	struct xfs_mount	*mp = ip->i_mount;
	// 将字节偏移量转换为文件系统块 (FSB) 偏移量
	xfs_fileoff_t		offset_fsb = XFS_B_TO_FSBT(mp, offset);
	// 计算写入范围末端的 FSB (考虑文件系统块大小)
	xfs_fileoff_t		end_fsb = xfs_iomap_end_fsb(mp, offset, count);
	// 用于存储 BMBT (B+tree Map Block Translation) 记录 (即 extent 信息)
	struct xfs_bmbt_irec	imap, cmap; // imap: data fork, cmap: cow fork
	struct xfs_iext_cursor	icur, ccur; // 用于在 extent B+tree 中查找的游标 icur: data fork, ccur: cow fork
	xfs_fsblock_t		prealloc_blocks = 0; // 要预分配的块数
	bool			eof = false, cow_eof = false, shared = false; // 状态标志
	// eof: data fork 查找是否到达末尾 (即 hole)
	// cow_eof: cow fork 查找是否到达末尾
	// shared: 找到的 extent 是否是 reflink 共享的
	int			allocfork = XFS_DATA_FORK; // 默认在数据 fork 中分配
	int			error = 0; // 错误码
	unsigned int		lockmode = XFS_ILOCK_EXCL; // 请求的 inode 锁模式
	unsigned int		iomap_flags = 0; // 要设置在 iomap 结构中的标志 (如 IOMAP_F_NEW)
	u64			seq; // inode 序列号，用于缓存一致性

	// 检查文件系统是否正在关闭
	if (xfs_is_shutdown(mp))
		return -EIO;

	// 如果设置了 extent 大小提示 (通常用于 Direct I/O)，则缓冲写入不适用
	// 转而调用 Direct I/O 的 iomap_begin 函数处理
	if (xfs_get_extsz_hint(ip))
		return xfs_direct_write_iomap_begin(inode, offset, count,
				flags, iomap, srcmap);

	// 附加磁盘配额信息 (如果启用了配额)
	error = xfs_qm_dqattach(ip);
	if (error)
		return error;

	// 获取 iomap 操作所需的 inode 锁 (通常是排他锁)
	error = xfs_ilock_for_iomap(ip, flags, &lockmode);
	if (error)
		return error;

	// 检查 inode 数据 fork 的 extent 格式是否损坏
	if (XFS_IS_CORRUPT(mp, !xfs_ifork_has_extents(&ip->i_df)) ||
	    XFS_TEST_ERROR(false, mp, XFS_ERRTAG_BMAPIFORMAT)) {
		xfs_bmap_mark_sick(ip, XFS_DATA_FORK); // 标记数据 fork 损坏
		error = -EFSCORRUPTED;
		goto out_unlock; // 跳转到解锁并返回错误
	}

	// 增加块映射写操作的统计计数器
	XFS_STATS_INC(mp, xs_blk_mapw);

	// 读取 inode 的数据 fork extent 到内存缓存中 (如果尚未加载)
	error = xfs_iread_extents(NULL, ip, XFS_DATA_FORK);
	if (error)
		goto out_unlock;

	/*
	 * 首先搜索数据 fork 以查找源映射。我们总是需要数据 fork 映射，
	 * 因为必须将其返回给 iomap 代码，以便上层写入代码可以读取数据
	 * 来执行未对齐写入的读-修改-写周期。
	 */
	// 在数据 fork (ip->i_df) 中查找覆盖 offset_fsb 的 extent
	// eof 为 true 表示在 offset_fsb 处没有找到 extent (即是一个 hole)
	// 找到的 extent 信息存储在 imap 中，查找游标在 icur
	eof = !xfs_iext_lookup_extent(ip, &ip->i_df, offset_fsb, &icur, &imap);
	if (eof)
		// 如果是 hole，暂时假设这个 hole 持续到请求范围的末尾
		imap.br_startoff = end_fsb;

	/*
	 * 对于 zeroing (清零) 或 unsharing (取消共享) 一个 hole 的情况，
	 * 我们永远不需要分配块。
	 */
	// 如果操作是 IOMAP_UNSHARE 或 IOMAP_ZERO，并且查找结果表明 offset_fsb 处
	// 确实是一个 hole (imap.br_startoff > offset_fsb)，
	// 那么目标范围已经是 hole 或未共享状态。
	if ((flags & (IOMAP_UNSHARE | IOMAP_ZERO)) &&
	    imap.br_startoff > offset_fsb) {
		// 创建一个表示 hole 的 iomap 映射
		xfs_hole_to_iomap(ip, iomap, offset_fsb, imap.br_startoff);
		goto out_unlock; // 处理完成，跳转去解锁并返回
	}

	/*
	 * 对于 zeroing 操作，修剪超出 EOF 块的 delalloc extent。
	 * 如果它起始于 EOF 块之外，则将其转换为 unwritten extent。
	 */
	// 如果是 IOMAP_ZERO 操作，并且找到了覆盖 offset_fsb 的 extent，
	// 并且这个 extent 是延迟分配的 (isnullstartblock)
	if ((flags & IOMAP_ZERO) && imap.br_startoff <= offset_fsb &&
	    isnullstartblock(imap.br_startblock)) {
		// 获取当前文件大小对应的 FSB
		xfs_fileoff_t eof_fsb = XFS_B_TO_FSB(mp, XFS_ISIZE(ip));

		// 如果清零操作从 EOF 或之后开始
		if (offset_fsb >= eof_fsb)
			// 跳转到 convert_delay，将延迟分配转换为 unwritten，无需分配
			goto convert_delay;
		// 如果清零范围超出了 EOF
		if (end_fsb > eof_fsb) {
			// 将范围末端裁剪到 EOF
			end_fsb = eof_fsb;
			// 相应地裁剪 imap 记录，避免为 EOF 之外的部分分配块
			xfs_trim_extent(&imap, offset_fsb,
					end_fsb - offset_fsb);
		}
	}

	/*
	 * 即使没有找到数据 fork extent，也要搜索 COW fork extent 列表。
	 * 这有两个目的：首先，这实现了使用 cowextsize 进行的推测性预分配，
	 * 这样我们也会取消共享与共享块相邻的块，而不仅仅是共享块本身。
	 * 其次，在 extent 列表中查找通常比去查找共享 extent 树要快。
	 */
	// 如果是 CoW inode (启用了 reflink)
	if (xfs_is_cow_inode(ip)) {
		// 如果 CoW fork 尚未初始化
		if (!ip->i_cowfp) {
			ASSERT(!xfs_is_reflink_inode(ip)); // 确认不是旧的 reflink inode
			xfs_ifork_init_cow(ip); // 初始化 CoW fork
		}
		// 在 CoW fork (ip->i_cowfp) 中查找覆盖 offset_fsb 的 extent
		// cow_eof 为 true 表示没找到
		// 找到的 extent 存入 cmap，游标为 ccur
		cow_eof = !xfs_iext_lookup_extent(ip, ip->i_cowfp, offset_fsb,
				&ccur, &cmap);
		// 如果在 CoW fork 中找到了覆盖 offset_fsb 的 extent
		if (!cow_eof && cmap.br_startoff <= offset_fsb) {
			// 追踪事件：找到了 CoW extent
			trace_xfs_reflink_cow_found(ip, &cmap);
			// 跳转到处理 CoW extent 的逻辑
			goto found_cow;
		}
	}

	// 如果数据 fork 查找找到了覆盖 offset_fsb 的 extent
	if (imap.br_startoff <= offset_fsb) {
		/*
		 * 对于 reflink 文件，当覆盖共享 extent 时，我们可能需要一个
		 * delalloc 预留。这也包括对包含数据的现有 extent 进行清零。
		 */
		// 如果不是 CoW inode，或者
		// 是 CoW inode 但正在 zeroing 一个非普通 (已写入) 的 extent (例如 delalloc 或 unwritten)
		// 这种情况下不需要 CoW 处理
		if (!xfs_is_cow_inode(ip) ||
		    ((flags & IOMAP_ZERO) && imap.br_state != XFS_EXT_NORM)) {
			// 追踪事件：找到了数据 fork extent
			trace_xfs_iomap_found(ip, offset, count, XFS_DATA_FORK,
					&imap);
			// 跳转到处理数据 fork extent 的逻辑
			goto found_imap;
		}

		// (走到这里说明是 CoW inode，并且覆盖的是普通 extent 或 hole)
		// 将找到的数据 extent (imap) 修剪到请求的范围 [offset_fsb, end_fsb)
		xfs_trim_extent(&imap, offset_fsb, end_fsb - offset_fsb);

		/*
		 * 将映射修剪到最近的共享 extent 边界。
		 * xfs_bmap_trim_cow 会查询 reflink B+树，检查 imap 描述的范围是否共享。
		 * 如果共享，它会设置 shared = true，并可能进一步修剪 imap 以匹配共享区域的边界。
		 */
		error = xfs_bmap_trim_cow(ip, &imap, &shared);
		if (error)
			goto out_unlock;

		// 如果检查后发现 extent 不共享
		if (!shared) {
			// 追踪事件：找到了非共享的数据 fork extent
			trace_xfs_iomap_found(ip, offset, count, XFS_DATA_FORK,
					&imap);
			// 跳转去处理这个 (可能被修剪过的) 数据 fork extent
			goto found_imap;
		}

		/*
		 * (走到这里说明 imap 描述的范围是共享的)
		 * Fork (取消共享) 从我们的写入偏移量开始直到 extent 末尾的所有共享块。
		 */
		allocfork = XFS_COW_FORK; // 标记需要在 CoW fork 中分配新块
		// 将分配范围的末端设置为找到的共享 extent 的末端
		// 这样可以确保整个共享区域都被 CoW 处理，而不仅仅是请求的 count 范围
		end_fsb = imap.br_startoff + imap.br_blockcount;
	} else { // (这个 else 对应 if (imap.br_startoff <= offset_fsb))
		/*
		 * (走到这里说明 offset_fsb 处是一个 hole，需要分配新块)
		 * 我们将此处映射的最大长度限制为 MAX_WRITEBACK_PAGES 页，
		 * 以保持完成的工作块大小与回写所做的工作大致对称。
		 * 这是一个完全凭空捏造的任意数字。
		 *
		 * 注意，在底层函数更新之前，该值需要小于 32 位宽。
		 */
		// 限制单次分配/映射的长度
		count = min_t(loff_t, count, 1024 * PAGE_SIZE);
		// 根据可能减小的 count 重新计算 end_fsb
		end_fsb = xfs_iomap_end_fsb(mp, offset, count);

		// 如果配置为总是 CoW (不常见)
		if (xfs_is_always_cow_inode(ip))
			allocfork = XFS_COW_FORK; // 在 CoW fork 分配
		// 否则 allocfork 保持默认值 XFS_DATA_FORK
	}

	// 如果是在文件末尾写入 (eof 为 true 且写入范围超出当前 ISIZE)
	if (eof && offset + count > XFS_ISIZE(ip)) {
		/*
		 * 确定预分配的初始大小。
		 * 我们在文件关闭时清理任何额外的预分配。
		 */
		// 根据挂载选项或启发式算法确定预分配块数
		if (xfs_has_allocsize(mp))
			prealloc_blocks = mp->m_allocsize_blocks;
		// 根据将在哪个 fork 分配，使用对应的游标来计算预分配大小
		else if (allocfork == XFS_DATA_FORK)
			prealloc_blocks = xfs_iomap_prealloc_size(ip, allocfork,
						offset, count, &icur);
		else // allocfork == XFS_COW_FORK
			prealloc_blocks = xfs_iomap_prealloc_size(ip, allocfork,
						offset, count, &ccur);

		// 如果需要预分配
		if (prealloc_blocks) {
			xfs_extlen_t	align; // 对齐要求
			xfs_off_t	end_offset; // 包含预分配的末尾偏移 (字节)
			xfs_fileoff_t	p_end_fsb; // 包含预分配的末尾 FSB

			// 计算包含预分配的末尾字节偏移，并按分配粒度对齐
			end_offset = XFS_ALLOC_ALIGN(mp, offset + count - 1);
			// 计算包含预分配的末尾 FSB
			p_end_fsb = XFS_B_TO_FSBT(mp, end_offset) +
					prealloc_blocks;

			// 获取 EOF 对齐要求
			align = xfs_eof_alignment(ip);
			if (align)
				// 如果有对齐要求，将预分配末尾向上舍入到对齐边界
				p_end_fsb = roundup_64(p_end_fsb, align);

			// 预分配末尾不能超过文件系统的最大文件大小
			p_end_fsb = min(p_end_fsb,
				XFS_B_TO_FSB(mp, mp->m_super->s_maxbytes));
			ASSERT(p_end_fsb > offset_fsb); // 确保预分配末尾在起始点之后
			// 计算实际需要添加的预分配块数 (总预分配块数 - 写入范围块数)
			prealloc_blocks = p_end_fsb - end_fsb;
		}
	}

	/*
	 * 用 IOMAP_F_NEW 标记新分配的 delalloc 块，以便在写入失败时
	 * 将它们打掉 (punch out)。
	 */
	iomap_flags |= IOMAP_F_NEW; // 标记为新分配 (延迟分配)

	// 如果需要在 CoW fork 中分配 (因为覆盖共享块)
	if (allocfork == XFS_COW_FORK) {
		// 在 CoW fork 中预留延迟分配空间
		// offset_fsb: 起始块
		// end_fsb - offset_fsb: 需要分配的块数 (覆盖共享区域)
		// prealloc_blocks: 额外的预分配块数
		// cmap: [输出] 生成的延迟分配 extent 记录
		// ccur: CoW fork 游标
		// cow_eof: CoW fork 查找是否到末尾
		error = xfs_bmapi_reserve_delalloc(ip, allocfork, offset_fsb,
				end_fsb - offset_fsb, prealloc_blocks, &cmap,
				&ccur, cow_eof);
		if (error) // 预留失败
			goto out_unlock;

		// 追踪事件：分配 CoW 延迟块
		trace_xfs_iomap_alloc(ip, offset, count, allocfork, &cmap);
		// 跳转到处理 CoW 映射的逻辑
		goto found_cow;
	}

	// (走到这里说明在 Data fork 中分配，可能是写入 hole 或非共享区域)
	// 在 Data fork 中预留延迟分配空间
	// imap: [输出] 生成的延迟分配 extent 记录
	// icur: Data fork 游标
	// eof: Data fork 查找是否到末尾 (是否在写 hole)
	error = xfs_bmapi_reserve_delalloc(ip, allocfork, offset_fsb,
			end_fsb - offset_fsb, prealloc_blocks, &imap, &icur,
			eof);
	if (error) // 预留失败
		goto out_unlock;

	// 追踪事件：分配 Data fork 延迟块
	trace_xfs_iomap_alloc(ip, offset, count, allocfork, &imap);
	// (自然落入 found_imap 逻辑)

// 处理数据 fork 映射的标签 (无论是找到的现有 extent 还是新分配的 delalloc extent)
found_imap:
	// 获取 inode 序列号，用于 iomap 缓存一致性
	seq = xfs_iomap_inode_sequence(ip, iomap_flags);
	// 释放 inode 锁
	xfs_iunlock(ip, lockmode);
	// 将 XFS 的 extent 记录 (imap) 转换为通用的 iomap 结构
	// flags: 原始传入的操作标志
	// iomap_flags: 新添加的标志 (如 IOMAP_F_NEW)
	// seq: inode 序列号
	return xfs_bmbt_to_iomap(ip, iomap, &imap, flags, iomap_flags, seq);

// 处理清零操作在 EOF 或之后开始的情况
convert_delay:
	// 释放 inode 锁
	xfs_iunlock(ip, lockmode);
	// 清理掉要转换范围内的页面缓存 (很重要，因为状态要变了)
	truncate_pagecache(inode, offset);
	// 将 offset 处的延迟分配 extent 转换为 unwritten extent
	// 这个函数会直接填充 iomap 结构，表示一个 unwritten 区域，并且不分配物理块
	error = xfs_bmapi_convert_delalloc(ip, XFS_DATA_FORK, offset,
					   iomap, NULL);
	if (error) // 转换失败
		return error;

	// 追踪事件 (虽然叫 alloc，但这里是 convert)
	trace_xfs_iomap_alloc(ip, offset, count, XFS_DATA_FORK, &imap);
	return 0; // 成功返回

// 处理 CoW fork 映射的标签 (新分配的用于覆盖共享块的 delalloc extent)
found_cow:
	// 如果原始数据 fork 查找确实在 offset_fsb 处找到了 extent (即 imap 有效)
	// 这意味着我们正在覆盖一个实际存在的 (可能是共享的) 数据区域
	if (imap.br_startoff <= offset_fsb) {
		// 将原始数据 extent (imap) 的信息填充到 srcmap 中
		// 上层 iomap 代码可能需要 srcmap 来执行读-修改-写操作
		error = xfs_bmbt_to_iomap(ip, srcmap, &imap, flags, 0,
				xfs_iomap_inode_sequence(ip, 0));
		if (error) // 转换失败
			goto out_unlock;
	} else { // 如果原始数据 fork 在 offset_fsb 处是 hole (imap 代表 hole 后面的第一个 extent)
		// 修剪新分配的 CoW extent (cmap)，使其末端不超过 imap 的起始位置
		// 这样可以确保新分配的块正好填充 hole，不会覆盖到后面的数据 extent
		xfs_trim_extent(&cmap, offset_fsb,
				imap.br_startoff - offset_fsb);
	}

	// 标记这个 iomap 映射涉及 CoW (即使新块本身未共享，但操作源于 CoW)
	iomap_flags |= IOMAP_F_SHARED;
	// 获取 inode 序列号
	seq = xfs_iomap_inode_sequence(ip, iomap_flags);
	// 释放 inode 锁
	xfs_iunlock(ip, lockmode);
	// 将新分配的 CoW extent 记录 (cmap) 转换为通用的 iomap 结构
	return xfs_bmbt_to_iomap(ip, iomap, &cmap, flags, iomap_flags, seq);

// 错误处理出口
out_unlock:
	// 释放 inode 锁
	xfs_iunlock(ip, lockmode);
	// 返回错误码
	return error;
}
```

总结一下 `xfs_buffered_write_iomap_begin` 的复杂性来源：

1.  **区分多种写入场景**：需要判断是覆盖已有数据、写入空洞、还是覆盖 reflink 共享数据。
2.  **处理 Reflink/CoW**：
    *   需要查找数据 fork 和 CoW fork。
    *   需要检测被覆盖的数据是否共享 (`xfs_bmap_trim_cow`)。
    *   如果覆盖共享数据，需要在 CoW fork 中进行延迟分配 (`allocfork = XFS_COW_FORK`)。
    *   需要填充 `srcmap` 以便上层进行 RMW。
3.  **延迟分配 (Delayed Allocation)**：缓冲写入的核心机制，不立即分配物理块，而是先预留空间并记录为 `delalloc` extent。实际分配推迟到页面回写时。`xfs_bmapi_reserve_delalloc` 是关键。
4.  **处理 Zeroing/Unsharing**：对于这些特殊操作，需要特殊处理，例如将 delalloc 转换为 unwritten (`convert_delay`) 或直接返回 hole 映射。
5.  **EOF 处理和预分配**：写入文件末尾时，需要计算并执行预分配 (`xfs_iomap_prealloc_size`, `prealloc_blocks`)，并考虑对齐 (`xfs_eof_alignment`)。
6.  **锁管理**：需要在操作开始时获取 inode 锁，并在返回映射信息 *之前* 释放锁，因为后续的数据拷贝 (`iomap_write_iter`) 不需要持有 inode 锁。
7.  **错误处理和状态跟踪**：需要仔细处理各种错误情况并正确管理各种状态标志和游标。

因为这个函数是根据很多具体情况来进不同分支的。虽然是一个大函数,但是我们可以代入不同的初始条件把函数逻辑拆解开来。
1. 假设我们不是cow inode,代码将缩减为:
* [xfs_buffered_write_iomap_begin_no_cow]()