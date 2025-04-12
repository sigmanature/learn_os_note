同时移除掉fallocate(也就是iomap zero部分)和cow之后的xfs_buffered_write_iomap_begin函数
```C
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
		imap.br_startoff = end_fsb;h
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
// 错误处理出口
out_unlock:
	// 释放 inode 锁
	xfs_iunlock(ip, lockmode);
	// 返回错误码
	return error;
}
```