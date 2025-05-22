```C
int f2fs_allocate_data_block(struct f2fs_sb_info *sbi, struct page *page,
		block_t old_blkaddr, block_t *new_blkaddr,
		struct f2fs_summary *sum, int type, /* 'type' is curseg_type from enum curseg_type */
		struct f2fs_io_info *fio)
{
	struct sit_info *sit_i = SIT_I(sbi); // 获取 Segment Information Table (SIT) 的信息结构
	struct curseg_info *curseg = CURSEG_I(sbi, type); // 根据传入的 'type' 获取对应的当前段 (CURSEG) 信息
	unsigned long long old_mtime; // 用于存储旧块的修改时间，以便在GC时继承
	bool from_gc = (type == CURSEG_ALL_DATA_ATGC); // 判断此分配是否由垃圾回收 (GC) 触发
	struct seg_entry *se = NULL; // 指向段条目 (Segment Entry)，用于GC场景
	bool segment_full = false; // 标记当前段是否在本次分配后已满
	int ret = 0;

	// 获取文件系统元数据锁 (SM_I(sbi)->curseg_lock)，保护对所有 curseg 信息的访问。
	// 这里使用读锁，因为主要目的是读取和选择一个 curseg，具体 curseg 的修改由其内部 mutex 保护。
	f2fs_down_read(&SM_I(sbi)->curseg_lock);

	// 锁定特定类型 (type) 的当前段 (curseg_mutex)，防止并发向同一个 curseg 分配块。
	mutex_lock(&curseg->curseg_mutex);
	// 写锁定 SIT 表项锁 (sentry_lock)，因为后续会更新 SIT（段的有效块计数、mtime等）。
	down_write(&sit_i->sentry_lock);

	// 检查选定的当前段是否有效 (即是否已分配了一个物理段号)。
	// 如果 curseg->segno 是 NULL_SEGNO，表示该类型的当前段尚未初始化或之前的段已用尽且未能分配新段。
	if (curseg->segno == NULL_SEGNO) {
		ret = -ENOSPC; // 返回“无可用空间”错误
		goto out_err;  // 跳转到错误处理并返回
	}

	// 如果分配请求来自垃圾回收 (GC)
	if (from_gc) {
		// GC 时，old_blkaddr 必然指向一个有效的、包含数据的旧块。
		f2fs_bug_on(sbi, GET_SEGNO(sbi, old_blkaddr) == NULL_SEGNO);
		// 获取旧块所在段的段条目信息。
		se = get_seg_entry(sbi, GET_SEGNO(sbi, old_blkaddr));
		// 检查段类型的合法性，确保 se->type 是一个有效的段类型。
		sanity_check_seg_type(sbi, se->type);
		// GC 操作应该针对数据段，而非节点段。
		f2fs_bug_on(sbi, IS_NODESEG(se->type));
	}
	// 计算新分配块的物理地址：当前段的起始地址 + 当前段内下一个可用的块偏移量。
	*new_blkaddr = NEXT_FREE_BLKADDR(sbi, curseg);

	// 断言检查：确保计算出的下一个块偏移量没有超出段内的总块数。
	f2fs_bug_on(sbi, curseg->next_blkoff >= BLKS_PER_SEG(sbi));

	// 如果设备支持或需要 discard/trim，等待针对新块地址的 discard 操作完成。
	// 这是为了确保在写入新数据前，该物理位置的旧数据（如果存在且已被标记为可擦除）已被处理。
	f2fs_wait_discard_bio(sbi, *new_blkaddr);

	// 将当前块的摘要信息 (sum) 记录到当前段的摘要块 (summary block) 的相应条目中。
	// 摘要信息包含了指向该数据块的元数据信息（如节点ID (nid)、节点内偏移 (ofs_in_node) 和版本号）。
	curseg->sum_blk->entries[curseg->next_blkoff] = *sum;

	// 根据当前段的分配类型 (alloc_type) 更新下一个可用块的偏移 (next_blkoff)。
	if (curseg->alloc_type == SSR) { // SSR (Static/Segment-level Sequential Remapping) 模式
		// SSR 模式下，会尝试寻找段内下一个逻辑上“更合适”的块，试图将逻辑连续的数据物理上也连续存放。
		curseg->next_blkoff = f2fs_find_next_ssr_block(sbi, curseg);
	} else { // LFS (Log-structured File System) 模式或 FSR (Fragment-level Sequential Remapping)
		curseg->next_blkoff++; // 简单地递增块偏移量。
		// 如果启用了 FS_MODE_FRAGMENT_BLK 选项 (试图在 chunk 级别随机化块分配以分散磨损)。
		if (F2FS_OPTION(sbi).fs_mode == FS_MODE_FRAGMENT_BLK)
			f2fs_randomize_chunk(sbi, curseg); // 对 chunk 内的块分配进行随机化处理。
	}

	// 检查当前段在分配完这个块之后是否已满。
	if (curseg->next_blkoff >= f2fs_usable_blks_in_seg(sbi, curseg->segno))
		segment_full = true;

	// 增加文件系统的总块分配计数 (用于统计信息)。
	stat_inc_block_count(sbi, curseg);

	// 更新段的修改时间 (mtime)。mtime 用于 GC 决策，通常反映了段中数据的“年龄”。
	if (from_gc) {
		// 如果是 GC，新块的 mtime 应该继承自被 GC 的旧块所在段的 mtime，以保持数据年龄信息。
		old_mtime = get_segment_mtime(sbi, old_blkaddr);
	} else {
		// 如果不是 GC (即常规写入新数据或覆盖写)，将旧块地址 (如果有效) 的 mtime 更新为0，
		// 表示旧数据不再活跃或已被迁移。
		update_segment_mtime(sbi, old_blkaddr, 0);
		old_mtime = 0; // 对于新分配的块，其 mtime 将基于当前时间 (在 update_segment_mtime 内部处理)。
	}
	// 更新新分配块所在段的 mtime。如果 old_mtime 非0 (来自GC)，则新段的 mtime 会考虑这个旧时间。
	update_segment_mtime(sbi, *new_blkaddr, old_mtime);

	/*
	 * SIT (Segment Information Table) 信息应该在段分配之前更新，
	 * 因为 SSR (Static/Segment-level Sequential Remapping) 需要最新的有效块信息来做决策。
	 * (更准确地说，是在下一个块偏移确定后，但在可能切换到新段之前)
	 */
	// 更新新块地址所在段的 SIT 条目：该段的有效块计数 (valid_map) 增加1。
	update_sit_entry(sbi, *new_blkaddr, 1);
	// 更新旧块地址所在段的 SIT 条目：该段的有效块计数减少1 (如果是 out-of-place update，旧块变无效)。
	// 如果 old_blkaddr 是 NULL_ADDR (例如，新文件首次分配块)，此操作对 SIT 无影响。
	update_sit_entry(sbi, old_blkaddr, -1);

	/*
	 * 如果当前段在分配此块后已满，则需要关闭当前段 (将其摘要页写回)，
	 * 并为后续写操作准备一个新的段。
	 */
	if (segment_full) {
		// 特殊处理：如果是固定的冷数据段 (CURSEG_COLD_DATA_PINNED)
		// 并且当前段是其所属 section 中的最后一个段。
		// "Pinned" 段类型用于确保某些类型的数据（如 fsync 密集型数据）有预留空间。
		// 当 section 的最后一个 pinned 段写满时，只写回其摘要页并重置字段，
		// 而不立即分配新段，以避免在 section 边界频繁地分配和关闭段。
		if (type == CURSEG_COLD_DATA_PINNED &&
		    !((curseg->segno + 1) % sbi->segs_per_sec)) { // 判断是否是 section 的最后一个段
			// 将当前段的摘要块 (curseg->sum_blk) 写回到磁盘上的永久摘要页位置。
			write_sum_page(sbi, curseg->sum_blk,
					GET_SUM_BLOCK(sbi, curseg->segno));
			// 重置当前段的内部字段 (如 next_blkoff, valid_blocks 等)，但不改变其 segno。
			// 这个段会暂时停止使用，直到下次明确需要这个类型的段时才可能被重新激活或替换。
			reset_curseg_fields(curseg);
			goto skip_new_segment; // 跳过分配新段的逻辑
		}

		if (from_gc) { // 如果是 GC 触发的段满
			// 为 GC 分配一个新的 ATSSR (Adaptive SSR) 段。
			// se->type 是被 GC 的段的原始类型，se->mtime 是其修改时间，这些信息用于指导新段的选择。
			ret = get_atssr_segment(sbi, type, se->type,
						AT_SSR, se->mtime);
		} else { // 如果是普通写入触发的段满
			if (need_new_seg(sbi, type)) // 判断是否真的需要分配一个全新的物理段
				ret = new_curseg(sbi, type, false); // 分配一个全新的段作为新的当前段
			else
				ret = change_curseg(sbi, type); // 尝试切换到同类型预留的下一个段 (如果 F2FS 支持并配置了这种机制)
			// 增加对应段类型的分配总数 (用于统计信息)
			stat_inc_seg_type(sbi, curseg);
		}

		if (ret) // 如果分配/切换新段失败 (例如，磁盘空间不足)
			goto out_err; // 跳转到错误处理路径
	}

skip_new_segment: // 跳过新段分配逻辑的标签

	/*
	 * 段的脏状态 (dirty status) 应该在段分配操作之后更新。
	 * 这样可以确保我们只在段实际被修改后才标记它，并且在旧段关闭、新段启用后，
	 * 两个相关段的脏状态都得到正确反映。
	 */
	// 将旧块地址所在段（如果有效）标记为脏段 (如果它之前不是脏的)。
	locate_dirty_segment(sbi, GET_SEGNO(sbi, old_blkaddr));
	// 将新块地址所在段标记为脏段。
	locate_dirty_segment(sbi, GET_SEGNO(sbi, *new_blkaddr));

	// 如果当前段是数据段 (而不是节点段，如 NAT/SIT 段)
	if (IS_DATASEG(curseg->seg_type))
		atomic64_inc(&sbi->allocated_data_blocks); // 原子增加文件系统中已分配数据块的总数。

	// 释放 SIT 表项锁。
	up_write(&sit_i->sentry_lock);

	// 如果当前写入的是节点页 (page != NULL，且当前段是节点段类型)。
	// 'page' 参数在 f2fs_write_node_page 时会传入，在 f2fs_write_data_page 时通常为 NULL (或不直接使用)。
	// (更正：对于数据页写，此处的 page 参数是 fio->page，但 IS_NODESEG 条件不会满足)
	// 此段逻辑主要用于写 NAT/SIT 等元数据节点页。
	if (page && IS_NODESEG(curseg->seg_type)) {
		// 在节点页的页脚部分填充下一个可用块的地址 (用于 F2FS 的 NAT/SIT journal 区域)。
		fill_node_footer_blkaddr(page, NEXT_FREE_BLKADDR(sbi, curseg));
		// 计算并设置节点页的校验和。
		f2fs_inode_chksum_set(sbi, page);
	}

	// 如果传入了 fio (f2fs_io_info) 结构体 (通常在数据页的 out-of-place 写路径中)。
	// fio 封装了关于本次 I/O 操作的各种信息。
	if (fio) {
		struct f2fs_bio_info *io; // 指向 sbi->write_io[fio->type] 中的一个元素

		INIT_LIST_HEAD(&fio->list); // 初始化 fio 结构体中的链表头 (struct list_head list)。
		fio->in_list = 1;           // 标记 fio 准备加入链表 (实际此时还未真正加入)。
		// 根据 fio 的类型 (fio->type 通常是 DATA 或 NODE) 和 fio->temp (用于多队列时的负载均衡)
		// 获取对应的 sbi->write_io[] 数组中的 f2fs_bio_info 结构。
		// sbi->write_io 是一个数组，每个元素代表一个 I/O 提交队列（包含一个 io_list）。
		io = sbi->write_io[fio->type] + fio->temp;
		spin_lock(&io->io_lock); // 锁定该 f2fs_bio_info 结构的自旋锁，以保护其内部的 io_list。
		list_add_tail(&fio->list, &io->io_list); // 将 fio 添加到选定 io_list 的尾部 (实现 FIFO)。
		spin_unlock(&io->io_lock); // 解锁。
	}

	mutex_unlock(&curseg->curseg_mutex); // 解锁当前段的互斥锁。
	f2fs_up_read(&SM_I(sbi)->curseg_lock); // 解锁 curseg_lock (读锁)。
	return 0; // 成功返回。

out_err: // 错误处理路径
	*new_blkaddr = NULL_ADDR; // 将新块地址设为无效。
	up_write(&sit_i->sentry_lock); // 释放 SIT 表项锁。
	mutex_unlock(&curseg->curseg_mutex); // 解锁当前段的互斥锁。
	f2fs_up_read(&SM_I(sbi)->curseg_lock); // 解锁 curseg_lock (读锁)。
	return ret; // 返回错误码。
}
```