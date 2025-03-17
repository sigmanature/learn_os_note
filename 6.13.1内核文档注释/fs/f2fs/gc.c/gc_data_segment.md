   好的，我们来深入剖析 `gc_data_segment` 函数及其调用链，理解 F2FS 如何进行数据段的垃圾回收，并详细分析你提供的所有代码片段。
**NAT相关宏：**[NAT_START_BLOCK_OFFSET](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.h/current_nat_addr.md)<br>
**1. `gc_data_segment(struct f2fs_sb_info *sbi, struct f2fs_summary *sum, struct gc_inode_list *gc_list, unsigned int segno, int gc_type, bool force_migrate)`**

```c
static int gc_data_segment(struct f2fs_sb_info *sbi, struct f2fs_summary *sum,
		struct gc_inode_list *gc_list, unsigned int segno, int gc_type,
		bool force_migrate)
{
	struct super_block *sb = sbi->sb;
	struct f2fs_summary *entry;
	block_t start_addr;
	int off;
	int phase = 0;
	int submitted = 0;
	unsigned int usable_blks_in_seg = f2fs_usable_blks_in_seg(sbi, segno);

	start_addr = START_BLOCK(sbi, segno);

next_step:
	entry = sum;

	for (off = 0; off < usable_blks_in_seg; off++, entry++) {
		struct page *data_page;
		struct inode *inode;
		struct node_info dni; /* dnode info for the data */
		unsigned int ofs_in_node, nofs;
		block_t start_bidx;
		nid_t nid = le32_to_cpu(entry->nid);

		/*
		 * stop BG_GC if there is not enough free sections.
		 * Or, stop GC if the segment becomes fully valid caused by
		 * race condition along with SSR block allocation.
		 */
		if ((gc_type == BG_GC && has_not_enough_free_secs(sbi, 0, 0)) ||
			(!force_migrate && get_valid_blocks(sbi, segno, true) ==
							CAP_BLKS_PER_SEC(sbi)))
			return submitted;

		if (check_valid_map(sbi, segno, off) == 0)
			continue;

		if (phase == 0) {
			f2fs_ra_meta_pages(sbi, NAT_BLOCK_OFFSET(nid), 1,
							META_NAT, true);
			continue;
		}

		if (phase == 1) {
			f2fs_ra_node_page(sbi, nid);
			continue;
		}

		/* Get an inode by ino with checking validity */
		if (!is_alive(sbi, entry, &dni, start_addr + off, &nofs))
			continue;

		if (phase == 2) {
			f2fs_ra_node_page(sbi, dni.ino);
			continue;
		}

		ofs_in_node = le16_to_cpu(entry->ofs_in_node);

		if (phase == 3) {
			int err;

			inode = f2fs_iget(sb, dni.ino);
			if (IS_ERR(inode))
				continue;

			if (is_bad_inode(inode) ||
					special_file(inode->i_mode)) {
				iput(inode);
				continue;
			}

			if (f2fs_has_inline_data(inode)) {
				iput(inode);
				set_sbi_flag(sbi, SBI_NEED_FSCK);
				f2fs_err_ratelimited(sbi,
					"inode %lx has both inline_data flag and "
					"data block, nid=%u, ofs_in_node=%u",
					inode->i_ino, dni.nid, ofs_in_node);
				continue;
			}

			err = f2fs_gc_pinned_control(inode, gc_type, segno);
			if (err == -EAGAIN) {
				iput(inode);
				return submitted;
			}

			if (!f2fs_down_write_trylock(
				&F2FS_I(inode)->i_gc_rwsem[WRITE])) {
				iput(inode);
				sbi->skipped_gc_rwsem++;
				continue;
			}

			start_bidx = f2fs_start_bidx_of_node(nofs, inode) +
								ofs_in_node;

			if (f2fs_meta_inode_gc_required(inode)) {
				int err = ra_data_block(inode, start_bidx);

				f2fs_up_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
				if (err) {
					iput(inode);
					continue;
				}
				add_gc_inode(gc_list, inode);
				continue;
			}

			data_page = f2fs_get_read_data_page(inode, start_bidx,
							REQ_RAHEAD, true, NULL);
			f2fs_up_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
			if (IS_ERR(data_page)) {
				iput(inode);
				continue;
			}

			f2fs_put_page(data_page, 0);
			add_gc_inode(gc_list, inode);
			continue;
		}

		/* phase 4 */
		inode = find_gc_inode(gc_list, dni.ino);
		if (inode) {
			struct f2fs_inode_info *fi = F2FS_I(inode);
			bool locked = false;
			int err;

			if (S_ISREG(inode->i_mode)) {
				if (!f2fs_down_write_trylock(&fi->i_gc_rwsem[WRITE])) {
					sbi->skipped_gc_rwsem++;
					continue;
				}
				if (!f2fs_down_write_trylock(
						&fi->i_gc_rwsem[READ])) {
					sbi->skipped_gc_rwsem++;
					f2fs_up_write(&fi->i_gc_rwsem[WRITE]);
					continue;
				}
				locked = true;

				/* wait for all inflight aio data */
				inode_dio_wait(inode);
			}

			start_bidx = f2fs_start_bidx_of_node(nofs, inode)
								+ ofs_in_node;
			if (f2fs_meta_inode_gc_required(inode))
				err = move_data_block(inode, start_bidx,
							gc_type, segno, off);
			else
				err = move_data_page(inode, start_bidx, gc_type,
								segno, off);

			if (!err && (gc_type == FG_GC ||
					f2fs_meta_inode_gc_required(inode)))
				submitted++;

			if (locked) {
				f2fs_up_write(&fi->i_gc_rwsem[READ]);
				f2fs_up_write(&fi->i_gc_rwsem[WRITE]);
			}

			stat_inc_data_blk_count(sbi, 1, gc_type);
		}
	}

	if (++phase < 5)
		goto next_step;

	return submitted;
}
```

*   **功能:** `gc_data_segment` 函数负责处理 **数据段** 的垃圾回收。  它遍历给定 segment (`segno`) 的 Summary Block (`sum`) 中的条目，检查每个数据块，并将有效的数据块迁移到新的位置，并更新父节点信息。 
*   **为什么gc_data_segment要有预读NAT,数据块的父亲node节点以及inode节点的阶段:** 因为垃圾回收和LFS普通的写操作不同。普通的LFS写操作因为是直接往文件系统末尾追加形式写入,所以并不需要事先知道原数据的数据块,而原数据块被存的地方只有两个,inode或者直接数据块。但是gc的时候,我们就得知道原数据块地址在哪了。我们手里有的只有summary_block的nid。那首先我们必须根据这些nid在NAT表中找到对应的node数据块的物理块地址。这就意味着我们必须把所有nid相关的NAT数据块读到内存中。接下来,我们要想获取summary_block中所有的真实数据块地址,就必须取得所有的node数据块,无论它是一个inode还是一个直接node数据块。由于我们事先是不知道这个数据segment中的数据块的父亲到底是一个inode还是一个直接node块,因此我们要把nid对应的所有数据块和从nid里拿到的inode号对应的块全部读到内存。
*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `struct f2fs_summary *sum`: 指向当前 segment 的 Summary Block 的指针。
    *   `struct gc_inode_list *gc_list`:  GC inode 列表，用于管理参与 GC 的 inode。
    *   `unsigned int segno`: 当前正在处理的 segment 号。
    *   `int gc_type`: 垃圾回收类型 (`FG_GC` 或 `BG_GC`)。
    *   `bool force_migrate`:  是否强制迁移数据块，即使 segment 不是最佳 victim。

*   **变量初始化:**  与 `gc_node_segment` 类似，初始化了各种局部变量，包括 `phase`, `submitted`, `usable_blks_in_seg`, `start_addr` 等。

*   **计算起始块地址 `start_addr = START_BLOCK(sbi, segno);`**:  计算当前 segment 的起始块地址。

*   **五阶段循环 (`next_step` 标签和 `phase` 变量):**  `gc_data_segment` 函数使用 `phase` 变量和 `goto next_step` 实现了 **五阶段** 的处理流程。
    *   **`next_step:` 标签:**  循环入口点。
    *   **`entry = sum;`:**  每次进入 `next_step`，`entry` 指针重置为 Summary Block 起始位置。
    *   **`for (off = 0; off < usable_blks_in_seg; off++, entry++) { ... }` 循环:**  遍历当前 segment 的 Summary Block 条目。
        *   **局部变量声明:**  声明 `data_page`, `inode`, `dni` (dnode info), `ofs_in_node`, `nofs`, `start_bidx`, `nid` 等局部变量。
        *   **后台 GC 停止条件和 Segment 完全有效性检查:**
            ```c
            if ((gc_type == BG_GC && has_not_enough_free_secs(sbi, 0, 0)) ||
                (!force_migrate && get_valid_blocks(sbi, segno, true) ==
                                        CAP_BLKS_PER_SEC(sbi)))
                return submitted;
            ```
            *   **后台 GC 停止条件:**  如果为后台 GC 且空闲 section 不足，则提前返回，停止后台 GC。
            *   **Segment 完全有效性检查:**  如果不是强制迁移 (`!force_migrate`) 且当前 segment 已经完全有效 (有效块数量等于 segment 容量)，则提前返回，停止 GC。  **避免 GC 无效 segment。**
        *   **块有效性检查:**  `if (check_valid_map(sbi, segno, off) == 0) continue;`:  检查当前块是否有效。  如果无效，则跳过。
        *   **`if (phase == 0) { ... }` (第一阶段):**  **NAT 元数据预读**，与 `gc_node_segment` Phase 0 相同。
        *   **`if (phase == 1) { ... }` (第二阶段):**  **节点页面预读**，与 `gc_node_segment` Phase 1 相同。
        *   **`if (!is_alive(sbi, entry, &dni, start_addr + off, &nofs)) continue;`:**  **检查数据块是否 "alive" (有效且可访问)**。  `is_alive` 函数会检查父节点是否有效，以及数据块地址是否与 Summary Block 中记录的地址一致。  如果数据块不是 alive，则跳过。
        *   **`if (phase == 2) { ... }` (第三阶段):**  **Inode 节点页面预读**。  `f2fs_ra_node_page(sbi, dni.ino);` 预读数据块所属 inode 的节点页面。
        *   **`ofs_in_node = le16_to_cpu(entry->ofs_in_node);`:**  从 Summary Block 条目中读取数据块在节点中的偏移量 `ofs_in_node`。
        *   **`if (phase == 3) { ... }` (第四阶段):**  **Inode 获取和数据页面预读/元数据 GC 判断**。
            ```c
            if (phase == 3) {
                int err;

                inode = f2fs_iget(sb, dni.ino); // 获取 inode
                if (IS_ERR(inode))
                    continue;

                // ... (inode 有效性检查, inline data 检查, pinned control) ...

                if (!f2fs_down_write_trylock(
                    &F2FS_I(inode)->i_gc_rwsem[WRITE])) { // 获取 inode GC 写锁
                    iput(inode);
                    sbi->skipped_gc_rwsem++;
                    continue;
                }

                start_bidx = f2fs_start_bidx_of_node(nofs, inode) +
                                                    ofs_in_node; // 计算数据块索引

                if (f2fs_meta_inode_gc_required(inode)) { // 元数据 GC 判断
                    int err = ra_data_block(inode, start_bidx); // 元数据 GC 预读

                    f2fs_up_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
                    if (err) {
                        iput(inode);
                        continue;
                    }
                    add_gc_inode(gc_list, inode); // 添加到 GC inode 列表
                    continue;
                }

                data_page = f2fs_get_read_data_page(inode, start_bidx,
                                        REQ_RAHEAD, true, NULL); // 数据页面预读
                f2fs_up_write(&F2FS_I(inode)->i_gc_rwsem[WRITE]);
                if (IS_ERR(data_page)) {
                    iput(inode);
                    continue;
                }

                f2fs_put_page(data_page, 0);
                add_gc_inode(gc_list, inode); // 添加到 GC inode 列表
                continue;
            }
            ```
            *   **获取 Inode:**  `inode = f2fs_iget(sb, dni.ino);`:  调用 `f2fs_iget` 函数 **根据 Inode 号获取 Inode 结构体**。
            *   **Inode 有效性检查:**  检查 Inode 是否有效 (`is_bad_inode`, `special_file`, `f2fs_has_inline_data`)。  如果无效，则释放 Inode 并跳过。
            *   **GC Pinned Control:**  `err = f2fs_gc_pinned_control(inode, gc_type, segno);`:  调用 `f2fs_gc_pinned_control` 函数 **检查 Inode 是否被 pinned (例如，正在进行 Direct IO)**。  如果被 pinned 且为后台 GC，则返回 `-EAGAIN`，提前返回，停止后台 GC。
            *   **获取 Inode GC 写锁:**  `if (!f2fs_down_write_trylock(&F2FS_I(inode)->i_gc_rwsem[WRITE])) { ... }`:  **尝试非阻塞地获取 Inode 的 GC 写信号量 (`i_gc_rwsem[WRITE]`)**。  如果获取失败，则释放 Inode 并跳过。  **Inode GC 读写信号量用于保护 Inode 的 GC 相关操作的并发安全。**
            *   **计算数据块索引:**  `start_bidx = f2fs_start_bidx_of_node(nofs, inode) + ofs_in_node;`:  调用 `f2fs_start_bidx_of_node` 函数 **计算数据块在 Inode 地址空间中的起始块索引 `start_bidx`**。
            *   **元数据 Inode GC 判断:**  `if (f2fs_meta_inode_gc_required(inode)) { ... }`:  **检查 Inode 是否需要进行元数据 GC (`f2fs_meta_inode_gc_required`)**。  元数据 Inode GC 可能用于回收 Inode 自身的元数据块。
                *   **元数据 GC 预读:**  `int err = ra_data_block(inode, start_bidx);`:  如果需要元数据 GC，则调用 `ra_data_block` 函数 **预读数据块**。  `ra_data_block` 函数可能针对元数据 GC 场景进行了优化预读。
                *   **添加到 GC Inode 列表:**  `add_gc_inode(gc_list, inode);`:  将 Inode 添加到 GC Inode 列表 `gc_list` 中，表示该 Inode 需要进行 GC 处理。
            *   **数据页面预读:**  `data_page = f2fs_get_read_data_page(inode, start_bidx, REQ_RAHEAD, true, NULL);`:  如果不需要元数据 GC，则调用 `f2fs_get_read_data_page` 函数 **预读数据页面**。  `REQ_RAHEAD` 标志表示预读操作。
            *   **添加到 GC Inode 列表:**  `add_gc_inode(gc_list, inode);`:  将 Inode 添加到 GC Inode 列表 `gc_list` 中。
        *   **`if (phase == 4) { ... }` (第五阶段):**  **数据块迁移**。
            ```c
            /* phase 4 */
            inode = find_gc_inode(gc_list, dni.ino); // 从 GC inode 列表查找 inode
            if (inode) {
                struct f2fs_inode_info *fi = F2FS_I(inode);
                bool locked = false;
                int err;

                if (S_ISREG(inode->i_mode)) { // Regular 文件特殊处理
                    if (!f2fs_down_write_trylock(&fi->i_gc_rwsem[WRITE])) { // 尝试获取 inode GC 写锁
                        sbi->skipped_gc_rwsem++;
                        continue;
                    }
                    if (!f2fs_down_write_trylock(
                            &fi->i_gc_rwsem[READ])) { // 尝试获取 inode GC 读锁
                        sbi->skipped_gc_rwsem++;
                        f2fs_up_write(&fi->i_gc_rwsem[WRITE]);
                        continue;
                    }
                    locked = true;

                    /* wait for all inflight aio data */
                    inode_dio_wait(inode); // 等待 Direct IO 完成
                }

                start_bidx = f2fs_start_bidx_of_node(nofs, inode)
                                                    + ofs_in_node; // 计算数据块索引
                if (f2fs_meta_inode_gc_required(inode)) // 元数据 GC 判断
                    err = move_data_block(inode, start_bidx,
                                gc_type, segno, off); // 元数据 GC 数据块迁移
                else
                    err = move_data_page(inode, start_bidx, gc_type,
                                    segno, off); // 数据页面迁移

                if (!err && (gc_type == FG_GC ||
                        f2fs_meta_inode_gc_required(inode)))
                    submitted++; // 统计提交的 IO 数量

                if (locked) { // 释放 inode GC 读写锁
                    f2fs_up_write(&fi->i_gc_rwsem[READ]);
                    f2fs_up_write(&fi->i_gc_rwsem[WRITE]);
                }

                stat_inc_data_blk_count(sbi, 1, gc_type); // 更新数据块统计信息
            }
            ```
            *   **从 GC Inode 列表查找 Inode:**  `inode = find_gc_inode(gc_list, dni.ino);`:  从 GC Inode 列表 `gc_list` 中查找 Inode 号为 `dni.ino` 的 Inode 结构体。  **Phase 4 只处理在 Phase 3 中添加到 GC Inode 列表中的 Inode。**
            *   **Regular 文件特殊处理 (锁和 Direct IO 等待):**  如果 Inode 是 Regular 文件 (`S_ISREG(inode->i_mode)`)，则尝试获取 Inode 的 GC 写锁和读锁，并等待 Direct IO 完成 (`inode_dio_wait(inode)`)。  **Regular 文件可能需要更严格的并发控制和同步机制，以保证数据一致性。**
            *   **计算数据块索引:**  `start_bidx = f2fs_start_bidx_of_node(nofs, inode) + ofs_in_node;`:  计算数据块索引。
            *   **数据块迁移 (根据元数据 GC 判断选择迁移函数):**
                *   `if (f2fs_meta_inode_gc_required(inode)) err = move_data_block(inode, start_bidx, gc_type, segno, off);`:  如果需要元数据 GC，则调用 `move_data_block` 函数 **迁移数据块 (可能涉及元数据块的特殊处理)**。
                *   `else err = move_data_page(inode, start_bidx, gc_type, segno, off);`:  如果不需要元数据 GC，则调用 `move_data_page` 函数 **迁移数据页面 (普通数据块迁移)**。
            *   **统计提交的 IO 数量:**  `if (!err && (gc_type == FG_GC || f2fs_meta_inode_gc_required(inode))) submitted++;`:  如果数据块迁移成功 (没有错误) 且为前台 GC 或元数据 GC，则增加 `submitted` 计数器。
            *   **释放 Inode GC 读写锁 (如果已获取):**  如果之前获取了 Inode GC 读写锁 (`locked` 为真)，则释放锁。
            *   **更新数据块统计信息:**  `stat_inc_data_blk_count(sbi, 1, gc_type);`:  增加数据块计数统计。

    *   **`if (++phase < 5) goto next_step;`:**  在遍历完当前 segment 的所有块后，`phase` 递增。  如果 `phase` 小于 5，则跳转回 `next_step` 标签，开始下一个阶段的处理。  **`gc_data_segment` 函数总共执行五个阶段 (phase 0, 1, 2, 3, 4)。**

*   **返回值:**  `return submitted;`:  返回 `submitted` 计数器，表示在前台 GC 或元数据 GC 中成功迁移的数据块数量。

*   **总结 `gc_data_segment`:**  `gc_data_segment` 函数通过 **五阶段处理** 和 **遍历 Summary Block 条目** 的方式，实现了数据段的垃圾回收。  **五个阶段分别负责 NAT 预读、节点页面预读、Inode 节点页面预读、Inode 获取和数据页面预读/元数据 GC 判断，以及实际的数据块迁移。**  函数内部进行了多重检查和并发控制，包括块有效性检查、Inode 有效性检查、GC Pinned Control、Inode GC 读写锁等，确保数据迁移的正确性和可靠性。  **实际的数据迁移操作由 `move_data_block` 和 `move_data_page` 函数执行，我们将在后续分析中详细研究这两个函数。**

**2. `f2fs_start_bidx_of_node(unsigned int node_ofs, struct inode *inode)`**
* [f2fs_start_bidx_of_node](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/f2fs_start_bidx_of_node.md)<br>

**3. `ra_data_block(struct inode *inode, pgoff_t index)`**
* [ra_data_block](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/ra_data_block.md)<br>

**4. `f2fs_get_read_data_page(struct inode *inode, pgoff_t index, blk_opf_t op_flags, bool for_write, pgoff_t *next_pgofs)`**
* [f2fs_get_read_data_page](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/f2fs_get_read_data_page.md)<br>

**5. `move_data_block(struct inode *inode, block_t bidx, int gc_type, unsigned int segno, int off)`**
* [move_data_block](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/move_data_block.md)<br>

**6. `move_data_page(struct inode *inode, block_t bidx, int gc_type, unsigned int segno, int off)`**
* [move_data_page](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/move_data_page.md)<br>


**总结 `gc_data_segment` 和相关函数分析:**

通过对 `gc_data_segment` 函数及其调用链的详细分析，我们可以看到 F2FS 数据段垃圾回收的复杂性和多样性。  `gc_data_segment` 函数通过 **五阶段处理**，实现了数据段的垃圾回收流程，包括 **元数据预读、节点页面预读、Inode 获取、数据页面预读和数据块迁移** 等关键步骤。  **`move_data_block` 和 `move_data_page` 函数是数据块迁移的核心函数，分别实现了块级别和页面级别的迁移策略，适用于不同的场景 (例如，元数据 Inode GC vs. 普通数据 GC)。**  `gc_data_segment` 函数内部进行了 **多重检查、并发控制和错误处理**，保证了数据段垃圾回收的 **可靠性、一致性和性能**。

接下来，我们可以继续深入探讨 Folio 优化在 `gc_data_segment` 函数中的应用，以及如何针对不同的阶段和函数选择合适的 Folio 优化策略。



