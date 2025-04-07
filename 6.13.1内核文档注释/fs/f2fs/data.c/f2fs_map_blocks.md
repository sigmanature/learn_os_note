**相关函数:**
* [f2fs_get_dnode_of_data](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_get_dnode_of_data.md)
* [f2fs_map_blocks_cached](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_map_blocks_cached.md)
* [f2fs_read_single_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_read_single_page.md)
```c
int f2fs_map_blocks(struct inode *inode, struct f2fs_map_blocks *map, int flag)
{
	unsigned int maxblocks = map->m_len;
	struct dnode_of_data dn;
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	int mode = map->m_may_create ? ALLOC_NODE : LOOKUP_NODE;/*进行读操作的时候肯定是后者*/
	pgoff_t pgofs, end_offset, end;
	int err = 0, ofs = 1;
	unsigned int ofs_in_node, last_ofs_in_node;
	blkcnt_t prealloc;
	block_t blkaddr;
	unsigned int start_pgofs;
	int bidx = 0;
	bool is_hole;

	if (!maxblocks)
		return 0;

	if (!map->m_may_create && f2fs_map_blocks_cached(inode, map, flag))
		goto out;

	map->m_bdev = inode->i_sb->s_bdev;
	map->m_multidev_dio =
		f2fs_allow_multi_device_dio(F2FS_I_SB(inode), flag);

	map->m_len = 0;/*标志实际映射的块数量。这个变量其实类似xfs_bmbt_irec中的block数量*/
	map->m_flags = 0;

	/* it only supports block size == page size */
	pgofs =	(pgoff_t)map->m_lblk; /*从f2fs_read_single_page传进来的起始的逻辑索引*/

next_dnode:
	if (map->m_may_create)
		f2fs_map_lock(sbi, flag);

	/* When reading holes, we need its node page */
	set_new_dnode(&dn, inode, NULL, NULL, 0);/*先拿到node页*/
	err = f2fs_get_dnode_of_data(&dn, pgofs, mode);
	if (err) {
		if (flag == F2FS_GET_BLOCK_BMAP)
			map->m_pblk = 0;
		if (err == -ENOENT)
			err = f2fs_map_no_dnode(inode, map, &dn, pgofs);
		goto unlock_out;
	}

	start_pgofs = pgofs;/*start_pgofs是循环的初始计数器*/
	prealloc = 0;
	last_ofs_in_node = ofs_in_node = dn.ofs_in_node;
	end_offset = ADDRS_PER_PAGE(dn.node_page, inode);
	/*循环开始*/
next_block:
	blkaddr = f2fs_data_blkaddr(&dn);/*然后拿到数据块地址*/
	is_hole = !__is_valid_data_blkaddr(blkaddr);/*空洞这里记录的是数据块有效的取反*/
	if (!is_hole &&
	    !f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE)) {
		err = -EFSCORRUPTED;/*既不是空洞 又不是有效块
		那肯定是出现文件系统污染了*/
		goto sync_out;
	}

	/* use out-place-update for direct IO under LFS mode */
	/*direct io 新地更新逻辑开始*/
	if (map->m_may_create && (is_hole ||
		(flag == F2FS_GET_BLOCK_DIO && f2fs_lfs_mode(sbi) &&
		!f2fs_is_pinned_file(inode)))) {
		if (unlikely(f2fs_cp_error(sbi))) {
			err = -EIO;
			goto sync_out;
		}

		switch (flag) {
		case F2FS_GET_BLOCK_PRE_AIO:
			if (blkaddr == NULL_ADDR) {
				prealloc++;
				last_ofs_in_node = dn.ofs_in_node;
			}
			break;
		case F2FS_GET_BLOCK_PRE_DIO:
		case F2FS_GET_BLOCK_DIO:
			err = __allocate_data_block(&dn, map->m_seg_type);
			if (err)
				goto sync_out;
			if (flag == F2FS_GET_BLOCK_PRE_DIO)
				file_need_truncate(inode);
			set_inode_flag(inode, FI_APPEND_WRITE);
			break;
		default:
			WARN_ON_ONCE(1);
			err = -EIO;
			goto sync_out;
		}

		blkaddr = dn.data_blkaddr;
		if (is_hole)
			map->m_flags |= F2FS_MAP_NEW;
	} else if (is_hole) {
		if (f2fs_compressed_file(inode) &&
		    f2fs_sanity_check_cluster(&dn)) {
			err = -EFSCORRUPTED;
			f2fs_handle_error(sbi,
					ERROR_CORRUPTED_CLUSTER);
			goto sync_out;
		}

		switch (flag) {
		case F2FS_GET_BLOCK_PRECACHE:
			goto sync_out;
		case F2FS_GET_BLOCK_BMAP:
			map->m_pblk = 0;
			goto sync_out;
		case F2FS_GET_BLOCK_FIEMAP:
			if (blkaddr == NULL_ADDR) {
				if (map->m_next_pgofs)
					*map->m_next_pgofs = pgofs + 1;
				goto sync_out;
			}
			break;
		case F2FS_GET_BLOCK_DIO:
			if (map->m_next_pgofs)
				*map->m_next_pgofs = pgofs + 1;
			break;
		default:
			/* for defragment case */
			if (map->m_next_pgofs)
				*map->m_next_pgofs = pgofs + 1;
			goto sync_out;
		}
	}
	/*异地更新逻辑结束*/
	if (flag == F2FS_GET_BLOCK_PRE_AIO)
		goto skip;/*实际上就是continue*/

	if (map->m_multidev_dio)
		bidx = f2fs_target_device_index(sbi, blkaddr);

	if (map->m_len == 0) { /*如果m_len还是处在刚初始化的阶段*/
		/* reserved delalloc block should be mapped for fiemap. */
		if (blkaddr == NEW_ADDR)
			map->m_flags |= F2FS_MAP_DELALLOC;
		/* DIO READ and hole case, should not map the blocks. 为文件空洞的时候不建立映射?*/
		if (!(flag == F2FS_GET_BLOCK_DIO && is_hole && !map->m_may_create))
			map->m_flags |= F2FS_MAP_MAPPED;

		map->m_pblk = blkaddr;
		map->m_len = 1;

		if (map->m_multidev_dio)
			map->m_bdev = FDEV(bidx).bdev;
	} else if (map_is_mergeable(sbi, map, blkaddr, flag, bidx, ofs)) {/*重点代码,f2fs会尝试进行块映射合并,对读来说最重要*/
		ofs++;
		map->m_len++;
	} else {
		goto sync_out;
	}

skip:
	dn.ofs_in_node++;
	pgofs++;

	/* preallocate blocks in batch for one dnode page */
	/*预分配逻辑 很大概率对我们处理回写逻辑的时候很有帮助*/
	if (flag == F2FS_GET_BLOCK_PRE_AIO &&
			(pgofs == end || dn.ofs_in_node == end_offset)) {

		dn.ofs_in_node = ofs_in_node;
		err = f2fs_reserve_new_blocks(&dn, prealloc);
		if (err)
			goto sync_out;

		map->m_len += dn.ofs_in_node - ofs_in_node;
		if (prealloc && dn.ofs_in_node != last_ofs_in_node + 1) {
			err = -ENOSPC;
			goto sync_out;
		}
		dn.ofs_in_node = end_offset;
	}

	if (flag == F2FS_GET_BLOCK_DIO && f2fs_lfs_mode(sbi) &&
	    map->m_may_create) {
		/* the next block to be allocated may not be contiguous. */
		if (GET_SEGOFF_FROM_SEG0(sbi, blkaddr) % BLKS_PER_SEC(sbi) ==
		    CAP_BLKS_PER_SEC(sbi) - 1)
			goto sync_out;
	}

	if (pgofs >= end)
		goto sync_out;
	else if (dn.ofs_in_node < end_offset)
		goto next_block;/*循环判断和跳转*/
	/*后处理逻辑开始*/
	if (flag == F2FS_GET_BLOCK_PRECACHE) {
		if (map->m_flags & F2FS_MAP_MAPPED) {
			unsigned int ofs = start_pgofs - map->m_lblk;

			f2fs_update_read_extent_cache_range(&dn,
				start_pgofs, map->m_pblk + ofs,
				map->m_len - ofs);
		}
	}

	f2fs_put_dnode(&dn);

	if (map->m_may_create) {
		f2fs_map_unlock(sbi, flag);
		f2fs_balance_fs(sbi, dn.node_changed);
	}
	goto next_dnode;

sync_out:

	if (flag == F2FS_GET_BLOCK_DIO && map->m_flags & F2FS_MAP_MAPPED) {
		/*
		 * for hardware encryption, but to avoid potential issue
		 * in future
		 */
		f2fs_wait_on_block_writeback_range(inode,
						map->m_pblk, map->m_len);

		if (map->m_multidev_dio) {
			block_t blk_addr = map->m_pblk;

			bidx = f2fs_target_device_index(sbi, map->m_pblk);

			map->m_bdev = FDEV(bidx).bdev;
			map->m_pblk -= FDEV(bidx).start_blk;

			if (map->m_may_create)
				f2fs_update_device_state(sbi, inode->i_ino,
							blk_addr, map->m_len);

			f2fs_bug_on(sbi, blk_addr + map->m_len >
						FDEV(bidx).end_blk + 1);
		}
	}

	if (flag == F2FS_GET_BLOCK_PRECACHE) {
		if (map->m_flags & F2FS_MAP_MAPPED) {
			unsigned int ofs = start_pgofs - map->m_lblk;

			f2fs_update_read_extent_cache_range(&dn,
				start_pgofs, map->m_pblk + ofs,
				map->m_len - ofs);
		}
		if (map->m_next_extent)
			*map->m_next_extent = pgofs + 1;
	}
	f2fs_put_dnode(&dn);
unlock_out:
	if (map->m_may_create) {
		f2fs_map_unlock(sbi, flag);
		f2fs_balance_fs(sbi, dn.node_changed);
	}
out:
	trace_f2fs_map_blocks(inode, map, flag, err);
	return err;
}
```

*   **功能:** `f2fs_map_blocks` 函数是 F2FS 中 **块映射的核心函数**。  **`f2fs_map_blocks` 函数负责将 *连续的逻辑块号范围* 转换为 *物理块地址范围*，并将映射结果存储在 `f2fs_map_blocks` 结构体 `map` 中。**  **`f2fs_map_blocks` 函数是 F2FS 数据读取和写入路径上的关键函数，用于建立逻辑块号到物理块地址的映射关系。**

*   **参数:**
    *   `struct inode *inode`:  Inode 结构体指针，表示要进行块映射的文件所属的 Inode。
    *   `struct f2fs_map_blocks *map`:  `f2fs_map_blocks` 结构体指针，用于 **输入逻辑块号范围** 和 **输出物理块地址范围** 等块映射信息。
    *   `int flag`:  块映射标志，用于 **控制块映射的行为** (例如，是否允许创建新块，是否用于 Direct IO 等)。

*   **变量初始化:**  初始化局部变量 `maxblocks`, `dn`, `sbi`, `mode`, `pgofs`, `end_offset`, `end`, `err`, `ofs`, `ofs_in_node`, `last_ofs_in_node`, `prealloc`, `blkaddr`, `start_pgofs`, `bidx`, `is_hole`。

*   **检查 `maxblocks` (要映射的最大块数量):**  `if (!maxblocks) return 0;`:  **如果 `maxblocks` 为 0，表示不需要映射任何块，直接返回成功。**

*   **Extent Cache 查找 (快速路径):**
    ```c
    if (!map->m_may_create && f2fs_map_blocks_cached(inode, map, flag))
        goto out;
    ```
    *   **检查 `map->m_may_create` 标志**。  `map->m_may_create` 标志表示是否允许创建新块 (例如，写入操作)。  **如果 `map->m_may_create` 为假 (例如，读取操作)，则尝试使用 Extent Cache 快速查找块映射信息。**
    *   **调用 `f2fs_map_blocks_cached(inode, map, flag)` 函数在 Extent Cache 中查找块映射信息**。  **`f2fs_map_blocks_cached` 函数负责在 Extent Cache 中查找 *逻辑块号 `map->m_lblk`* 对应的 *物理块地址范围*，并将查找结果填充到 `map` 结构体中。**
    *   **`goto out;`:**  **如果 Extent Cache 命中，则 *直接跳转到 `out` 标签，返回成功*，复用 Extent Cache 的块映射结果。**  **Extent Cache 查找是块映射的 *快速路径*，可以提高性能。**

*   **初始化 `f2fs_map_blocks` 结构体:**
    ```c
    map->m_bdev = inode->i_sb->s_bdev;
    map->m_multidev_dio =
        f2fs_allow_multi_device_dio(F2FS_I_SB(inode), flag);

    map->m_len = 0;
    map->m_flags = 0;
    ```
    *   **设置 `map->m_bdev` 为 Inode 所在文件系统的块设备 `inode->i_sb->s_bdev`**。
    *   **设置 `map->m_multidev_dio` 标志，表示是否允许多设备 Direct IO (`f2fs_allow_multi_device_dio`)**。
    *   **初始化 `map->m_len` 为 0，`map->m_flags` 为 0**。  `map->m_len` 用于记录 *实际映射的块数量*，`map->m_flags` 用于记录块映射的 *状态标志*。

*   **初始化逻辑块号范围:**
    ```c
    /* it only supports block size == page size */
    pgofs =	(pgoff_t)map->m_lblk;
    end = pgofs + maxblocks;
    ```
    *   **设置 `pgofs` 为起始逻辑块号 `map->m_lblk`，`end` 为结束逻辑块号 (起始逻辑块号 + 最大块数量)。**  **`pgofs` 变量在后续循环中 *递增*，用于遍历要映射的逻辑块范围。**

*   **`next_dnode:` 标签和 Dnode 查找循环:**
    ```c
    next_dnode:
	if (map->m_may_create)
		f2fs_map_lock(sbi, flag);

	/* When reading holes, we need its node page */
	set_new_dnode(&dn, inode, NULL, NULL, 0);
	err = f2fs_get_dnode_of_data(&dn, pgofs, mode);
	if (err) {
		if (flag == F2FS_GET_BLOCK_BMAP)
			map->m_pblk = 0;
		if (err == -ENOENT)
			err = f2fs_map_no_dnode(inode, map, &dn, pgofs);
		goto unlock_out;
	}

	start_pgofs = pgofs;
	prealloc = 0;
	last_ofs_in_node = ofs_in_node = dn.ofs_in_node;
	end_offset = ADDRS_PER_PAGE(dn.node_page, inode);
    ```
    *   **`next_dnode:` 标签:**  **外层循环的入口标签，用于处理 *连续的 Dnode 范围*。**  **F2FS 的数据块地址映射是 *基于 Dnode* 的，一个 Dnode 页面可以管理 *多个* 数据块地址。**  **外层循环用于遍历 *不同的 Dnode 页面*。**
    *   **获取 Map Lock (如果需要):**  `if (map->m_may_create) f2fs_map_lock(sbi, flag);`:  **如果 `map->m_may_create` 为真 (例如，写入操作)，则获取 Map Lock (`f2fs_map_lock`)，保护块映射操作的并发安全。**
    *   **初始化 `dnode_of_data` 结构体 `dn`**。
    *   **调用 `f2fs_get_dnode_of_data(&dn, pgofs, mode)` 函数，根据 *逻辑块号 `pgofs`* 查找对应的 *Dnode 信息*，并将结果填充到 `dn` 结构体中**。  `mode` 参数根据 `map->m_may_create` 标志设置为 `ALLOC_NODE` (允许分配新节点) 或 `LOOKUP_NODE` (只查找节点)。  **`f2fs_get_dnode_of_data` 函数是块映射的核心函数，负责在 Inode 的 Dnode 树中查找逻辑块号对应的 Dnode 信息。**
    *   **错误处理:**  如果 `f2fs_get_dnode_of_data` 返回错误，则进行错误处理。  例如，如果错误为 `-ENOENT` (未找到 Dnode)，则调用 `f2fs_map_no_dnode` 函数处理 Hole (空洞) 情况。
    *   **初始化局部变量 `start_pgofs`, `prealloc`, `last_ofs_in_node`, `ofs_in_node`, `end_offset`**。  `start_pgofs` 记录当前 Dnode 页面的起始逻辑块号，`prealloc` 用于预分配计数，`ofs_in_node` 和 `last_ofs_in_node` 记录 Dnode 页面内的偏移量，`end_offset` 记录 Dnode 页面的结束偏移量。

*   **`next_block:` 标签和块地址遍历循环:**
    ```c
    next_block:
	blkaddr = f2fs_data_blkaddr(&dn);
	is_hole = !__is_valid_data_blkaddr(blkaddr);
	if (!is_hole &&
	    !f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE)) {
		err = -EFSCORRUPTED;
		goto sync_out;
	}

	// ... (Inplace Update, Outplace Update, Hole 处理逻辑) ...

	if (flag == F2FS_GET_BLOCK_PRE_AIO)
		goto skip;

	if (map->m_multidev_dio)
		bidx = f2fs_target_device_index(sbi, blkaddr);

	if (map->m_len == 0) {
		// ... (初始化块映射信息) ...
	} else if (map_is_mergeable(sbi, map, blkaddr, flag, bidx, ofs)) {
		// ... (合并块映射信息) ...
	} else {
		goto sync_out;
	}

    skip:
	dn.ofs_in_node++;
	pgofs++;

	// ... (预分配逻辑, DIO 特殊处理, 循环结束条件) ...

	if (pgofs >= end)
		goto sync_out;
	else if (dn.ofs_in_node < end_offset)
		goto next_block;

	// ... (Extent Cache 更新, Dnode 释放, Map Unlock, Balance FS, 下一个 Dnode) ...
    ```
    *   **`next_block:` 标签:**  **内层循环的入口标签，用于遍历 *当前 Dnode 页面* 管理的 *连续逻辑块范围*。**  **内层循环用于遍历 *同一个 Dnode 页面内的不同数据块地址*。**
    *   **获取数据块地址:**  `blkaddr = f2fs_data_blkaddr(&dn);`:  **调用 `f2fs_data_blkaddr(&dn)` 函数，从 `dnode_of_data` 结构体 `dn` 中 *获取当前逻辑块号 `pgofs` 对应的 *物理块地址 `blkaddr`***。  **`f2fs_data_blkaddr` 函数会根据 `dn` 结构体中的信息 (例如，`dn->node_page`, `dn->ofs_in_node`)，从 Dnode 页面中读取数据块地址。**
    *   **Hole 检查:**  `is_hole = !__is_valid_data_blkaddr(blkaddr);`:  **检查 `blkaddr` 是否为 Hole (空洞)**。  **Hole 表示逻辑块号 *没有* 对应的物理块地址，通常表示文件中的 *稀疏区域*。**
    *   **物理块地址有效性检查:**  `if (!is_hole && !f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE)) { err = -EFSCORRUPTED; goto sync_out; }`:  **如果 `blkaddr` 不是 Hole，则 *检查物理块地址 `blkaddr` 的有效性***。  如果无效，则返回 `-EFSCORRUPTED` 错误。
    *   **Inplace Update, Outplace Update, Hole 处理逻辑:**  **根据 `map->m_may_create` 标志和 `flag` 参数，以及 `is_hole` 变量，进行不同的处理**，包括 Inplace Update (原位更新), Outplace Update (异位更新), Hole 处理等。  **这部分代码比较复杂，涉及到 F2FS 的写操作和空间分配策略，需要仔细分析。**
    *   **`skip:` 标签:**  **`F2FS_GET_BLOCK_PRE_AIO` 预分配模式下，跳转到 `skip` 标签，跳过块映射信息更新和合并逻辑。**
    *   **多设备 Direct IO 处理:**  `if (map->m_multidev_dio) bidx = f2fs_target_device_index(sbi, blkaddr);`:  **如果允许多设备 Direct IO，则调用 `f2fs_target_device_index` 函数，根据物理块地址 `blkaddr` 获取 *目标设备索引 `bidx`***。  **多设备 F2FS 可以将文件数据分布在 *多个块设备* 上，提高 IO 并行度。**
    *   **初始化块映射信息 (`map->m_len == 0`)**:  **如果是 *首次映射* (`map->m_len == 0`)，则 *初始化 `map` 结构体的块映射信息***，包括：设置 `map->m_flags` (例如，`F2FS_MAP_MAPPED`, `F2FS_MAP_DELALLOC`)，设置 `map->m_pblk` (起始物理块地址)，设置 `map->m_len` (映射块数量)，设置 `map->m_bdev` (块设备)。
    *   **合并块映射信息 (`map_is_mergeable`)**:  **如果 *不是首次映射*，则 *检查当前块是否可以与之前的块映射信息进行 *合并* (`map_is_mergeable`)**。  合并条件包括：物理块地址是否连续，是否在同一个块设备上等。  **如果可以合并，则 *增加 `map->m_len`，扩展块映射范围*。**
    *   **`sync_out:` 标签 (无法合并):**  **如果无法合并，则跳转到 `sync_out` 标签，结束当前 Dnode 页面的块映射循环。**  **表示当前 Dnode 页面管理的连续逻辑块范围已经映射完成，需要处理下一个 Dnode 页面。**
    *   **`skip:` 标签:**  **`F2FS_GET_BLOCK_PRE_AIO` 预分配模式下，跳转到 `skip` 标签，跳过块映射信息更新和合并逻辑。**
    *   **更新 `dn->ofs_in_node` 和 `pgofs`:**  `dn.ofs_in_node++; pgofs++;`:  **递增 `dn->ofs_in_node` (Dnode 页面内偏移量) 和 `pgofs` (逻辑块号)，准备处理 *下一个逻辑块*。**
    *   **预分配逻辑 (`F2FS_GET_BLOCK_PRE_AIO`)**:  **`F2FS_GET_BLOCK_PRE_AIO` 预分配模式下的特殊处理**，例如，批量预分配块，预留空间等。
    *   **DIO 特殊处理 (`F2FS_GET_BLOCK_DIO && f2fs_lfs_mode(sbi) && map->m_may_create`)**:  **Direct IO 和 LFS 模式下的特殊处理**，例如，限制连续分配的块数量，避免跨越 Segment 边界。
    *   **循环结束条件:**  `if (pgofs >= end) goto sync_out; else if (dn.ofs_in_node < end_offset) goto next_block;`:  **内层循环的结束条件**。  如果 `pgofs` 达到结束逻辑块号 `end`，或者 `dn->ofs_in_node` 达到 Dnode 页面的结束偏移量 `end_offset`，则结束内层循环，跳转到 `sync_out` 标签。  否则，继续内层循环，处理下一个逻辑块 (`goto next_block`)。

*   **Extent Cache 更新 (预读模式 `F2FS_GET_BLOCK_PRECACHE`)**:
    ```c
    if (flag == F2FS_GET_BLOCK_PRECACHE) {
        if (map->m_flags & F2FS_MAP_MAPPED) {
            unsigned int ofs = start_pgofs - map->m_lblk;

            f2fs_update_read_extent_cache_range(&dn,
                start_pgofs, map->m_pblk + ofs,
                map->m_len - ofs);
        }
    }
    ```
    *   **检查是否为 `F2FS_GET_BLOCK_PRECACHE` 预读模式**。
    *   **如果块映射成功 (`map->m_flags & F2FS_MAP_MAPPED`)，则调用 `f2fs_update_read_extent_cache_range` 函数 *更新 Extent Cache*，将 *逻辑块号范围* (`start_pgofs` 到 `start_pgofs + map->m_len - ofs`) 和 *物理块地址范围* (`map->m_pblk + ofs` 到 `map->m_pblk + map->m_len - 1`) 的映射关系 *添加到 Extent Cache 中***。  **Extent Cache 更新可以加速后续的块映射操作。**

*   **Dnode 释放, Map Unlock, Balance FS, 下一个 Dnode:**
    ```c
    f2fs_put_dnode(&dn);

    if (map->m_may_create) {
        f2fs_map_unlock(sbi, flag);
        f2fs_balance_fs(sbi, dn.node_changed);
    }
    goto next_dnode;
    ```
    *   **释放 `dnode_of_data` 结构体 `dn` (`f2fs_put_dnode`)**。
    *   **释放 Map Lock (如果之前获取了) (`f2fs_map_unlock`)**。
    *   **调用 `f2fs_balance_fs` 函数进行文件系统平衡操作 (如果 Dnode 页面被修改了 `dn.node_changed`)**。  **文件系统平衡操作可能包括 Segment 整理、GC 等，用于提高文件系统性能和空间利用率。**
    *   **`goto next_dnode;`:**  **跳转到 `next_dnode` 标签，开始处理 *下一个 Dnode 页面* 的块映射。**

*   **`sync_out:` 标签 (同步输出):**
    ```c
    sync_out:
	if (flag == F2FS_GET_BLOCK_DIO && map->m_flags & F2FS_MAP_MAPPED) {
		/*
		 * for hardware encryption, but to avoid potential issue
		 * in future
		 */
		f2fs_wait_on_block_writeback_range(inode,
						map->m_pblk, map->m_len);

		if (map->m_multidev_dio) {
			block_t blk_addr = map->m_pblk;

			bidx = f2fs_target_device_index(sbi, map->m_pblk);

			map->m_bdev = FDEV(bidx).bdev;
			map->m_pblk -= FDEV(bidx).start_blk;

			if (map->m_may_create)
				f2fs_update_device_state(sbi, inode->i_ino,
							blk_addr, map->m_len);

			f2fs_bug_on(sbi, blk_addr + map->m_len >
						FDEV(bidx).end_blk + 1);
		}
	}

	if (flag == F2FS_GET_BLOCK_PRECACHE) {
		if (map->m_flags & F2FS_MAP_MAPPED) {
			unsigned int ofs = start_pgofs - map->m_lblk;

			f2fs_update_read_extent_cache_range(&dn,
				start_pgofs, map->m_pblk + ofs,
				map->m_len - ofs);
		}
		if (map->m_next_extent)
			*map->m_next_extent = pgofs + 1;
	}
	f2fs_put_dnode(&dn);
    unlock_out:
	if (map->m_may_create) {
		f2fs_map_unlock(sbi, flag);
		f2fs_balance_fs(sbi, dn.node_changed);
	}
    out:
	trace_f2fs_map_blocks(inode, map, flag, err);
	return err;
    }
    ```

*   **`sync_out:` 标签:**  **内层循环或外层循环结束时，跳转到 `sync_out` 标签，进行 *同步输出* 处理。**  **`sync_out` 标签下的代码主要负责 *清理工作* 和 *错误处理*。**
    *   **DIO Writeback 等待 (`F2FS_GET_BLOCK_DIO && map->m_flags & F2FS_MAP_MAPPED`)**:  **如果是 Direct IO 且块映射成功，则 *等待物理块范围的 Writeback 操作完成* (`f2fs_wait_on_block_writeback_range`)**。  **确保 Direct IO 的数据一致性。**
    *   **多设备 Direct IO 处理 (`map->m_multidev_dio`)**:  **如果是多设备 Direct IO，则进行多设备相关的处理**，例如，更新 `map->m_bdev` 和 `map->m_pblk`，更新设备状态 (`f2fs_update_device_state`)。
    *   **Extent Cache 更新 (预读模式 `F2FS_GET_BLOCK_PRECACHE`)**:  **如果是 `F2FS_GET_BLOCK_PRECACHE` 预读模式，则 *再次尝试更新 Extent Cache* (与 `next_block` 循环后的 Extent Cache 更新逻辑相同)。**  **可能用于处理循环过程中 Extent Cache 更新失败的情况。**
    *   **Dnode 释放 (`f2fs_put_dnode(&dn)`)**。

*   **`unlock_out:` 标签 (解锁输出):**  **`unlock_out` 标签用于 *解锁 Map Lock* 和 *Balance FS*，并跳转到 `out` 标签。**
    *   **Map Unlock (`f2fs_map_unlock`)**。
    *   **Balance FS (`f2fs_balance_fs`)**。

*   **`out:` 标签和返回值:**  `trace_f2fs_map_blocks(inode, map, flag, err); return err;`:  **记录 `f2fs_map_blocks` 函数的 tracepoint 信息，并返回错误码 `err`。**

*   **总结 `f2fs_map_blocks`:**  `f2fs_map_blocks` 函数实现了 **F2FS 块映射的核心逻辑**，包括 **Extent Cache 快速路径、Dnode 树查找、物理块地址获取、Hole 处理、Inplace/Outplace Update 选择、BIO 合并优化、多设备 Direct IO 支持、Extent Cache 更新和错误处理** 等复杂功能。  **`f2fs_map_blocks` 函数通过 *两层循环* (Dnode 循环和块循环) 遍历逻辑块范围，并根据 Inode 的元数据信息 (Dnode 树, Extent Cache) 将逻辑块号转换为物理块地址，最终将块映射信息存储在 `f2fs_map_blocks` 结构体中。**  **`f2fs_map_blocks` 函数是 F2FS 数据访问路径上的关键函数，体现了 F2FS 在地址映射、性能优化和功能丰富性方面的设计。**