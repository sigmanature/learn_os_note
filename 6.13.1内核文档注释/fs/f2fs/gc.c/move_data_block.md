
```c
static int move_data_block(struct inode *inode, block_t bidx,
				int gc_type, unsigned int segno, int off)
{
	struct address_space *mapping = f2fs_is_cow_file(inode) ?
				F2FS_I(inode)->atomic_inode->i_mapping : inode->i_mapping;
	struct f2fs_io_info fio = {
		.sbi = F2FS_I_SB(inode),
		.ino = inode->i_ino,
		.type = DATA,
		.temp = COLD,
		.op = REQ_OP_READ,
		.op_flags = 0,
		.encrypted_page = NULL,
		.in_list = 0,
	};
	struct dnode_of_data dn;
	struct f2fs_summary sum;
	struct node_info ni;
	struct page *page, *mpage;
	block_t newaddr;
	int err = 0;
	bool lfs_mode = f2fs_lfs_mode(fio.sbi);
	int type = fio.sbi->am.atgc_enabled && (gc_type == BG_GC) &&
				(fio.sbi->gc_mode != GC_URGENT_HIGH) ?
				CURSEG_ALL_DATA_ATGC : CURSEG_COLD_DATA;

	/* do not read out */
	page = f2fs_grab_cache_page(mapping, bidx, false);
	if (!page)
		return -ENOMEM;

	if (!check_valid_map(F2FS_I_SB(inode), segno, off)) {
		err = -ENOENT;
		goto out;
	}

	err = f2fs_gc_pinned_control(inode, gc_type, segno);
	if (err)
		goto out;

	set_new_dnode(&dn, inode, NULL, NULL, 0);
	err = f2fs_get_dnode_of_data(&dn, bidx, LOOKUP_NODE);
	if (err)
		goto out;

	if (unlikely(dn.data_blkaddr == NULL_ADDR)) {
		ClearPageUptodate(page);
		err = -ENOENT;
		goto put_out;
	}

	/*
	 * don't cache encrypted data into meta inode until previous dirty
	 * data were writebacked to avoid racing between GC and flush.
	 */
	f2fs_wait_on_page_writeback(page, DATA, true, true);

	f2fs_wait_on_block_writeback(inode, dn.data_blkaddr);

	err = f2fs_get_node_info(fio.sbi, dn.nid, &ni, false);
	if (err)
		goto put_out;

	/* read page */
	fio.page = page;
	fio.new_blkaddr = fio.old_blkaddr = dn.data_blkaddr;

	if (lfs_mode)
		f2fs_down_write(&fio.sbi->io_order_lock);

	mpage = f2fs_grab_cache_page(META_MAPPING(fio.sbi),
					fio.old_blkaddr, false);
	if (!mpage) {
		err = -ENOMEM;
		goto up_out;
	}

	fio.encrypted_page = mpage;

	/* read source block in mpage */
	if (!PageUptodate(mpage)) {
		err = f2fs_submit_page_bio(&fio);
		if (err) {
			f2fs_put_page(mpage, 1);
			goto up_out;
		}

		f2fs_update_iostat(fio.sbi, inode, FS_DATA_READ_IO,
							F2FS_BLKSIZE);
		f2fs_update_iostat(fio.sbi, NULL, FS_GDATA_READ_IO,
							F2FS_BLKSIZE);

		lock_page(mpage);
		if (unlikely(mpage->mapping != META_MAPPING(fio.sbi) ||
						!PageUptodate(mpage))) {
			err = -EIO;
			f2fs_put_page(mpage, 1);
			goto up_out;
		}
	}

	set_summary(&sum, dn.nid, dn.ofs_in_node, ni.version);

	/* allocate block address */
	err = f2fs_allocate_data_block(fio.sbi, NULL, fio.old_blkaddr, &newaddr,
				&sum, type, NULL);
	if (err) {
		f2fs_put_page(mpage, 1);
		/* filesystem should shutdown, no need to recovery block */
		goto up_out;
	}

	fio.encrypted_page = f2fs_pagecache_get_page(META_MAPPING(fio.sbi),
				newaddr, FGP_LOCK | FGP_CREAT, GFP_NOFS);
	if (!fio.encrypted_page) {
		err = -ENOMEM;
		f2fs_put_page(mpage, 1);
		goto recover_block;
	}

	/* write target block */
	f2fs_wait_on_page_writeback(fio.encrypted_page, DATA, true, true);
	memcpy(page_address(fio.encrypted_page),
				page_address(mpage), PAGE_SIZE);
	f2fs_put_page(mpage, 1);

	f2fs_invalidate_internal_cache(fio.sbi, fio.old_blkaddr);

	set_page_dirty(fio.encrypted_page);
	if (clear_page_dirty_for_io(fio.encrypted_page))
		dec_page_count(fio.sbi, F2FS_DIRTY_META);

	set_page_writeback(fio.encrypted_page);

	fio.op = REQ_OP_WRITE;
	fio.op_flags = REQ_SYNC;
	fio.new_blkaddr = newaddr;
	f2fs_submit_page_write(&fio);

	f2fs_update_iostat(fio.sbi, NULL, FS_GC_DATA_IO, F2FS_BLKSIZE);

	f2fs_update_data_blkaddr(&dn, newaddr);
	set_inode_flag(inode, FI_APPEND_WRITE);

	f2fs_put_page(fio.encrypted_page, 1);
recover_block:
	if (err)
		f2fs_do_replace_block(fio.sbi, &sum, newaddr, fio.old_blkaddr,
							true, true, true);
up_out:
	if (lfs_mode)
		f2fs_up_write(&fio.sbi->io_order_lock);
put_out:
	f2fs_put_dnode(&dn);
out:
	f2fs_put_page(page, 1);
	return err;
}
```

*   **功能:** `move_data_block` 函数用于 **迁移数据块 (block-level move)**。  **这个函数可能专门用于元数据 Inode GC 场景，直接在块设备层面上进行数据块的复制和迁移，而不是通过页面缓存。**  `move_data_block` 函数会将源数据块读取到元数据 mapping 的页面缓存中，然后分配新的数据块地址，将数据从元数据 mapping 页面复制到新的数据块地址，并更新元数据信息。

*   **参数:**
    *   `struct inode *inode`:  Inode 结构体指针，表示要迁移数据块所属的 Inode。
    *   `block_t bidx`:  要迁移的数据块在 Inode 地址空间中的块索引。
    *   `int gc_type`:  垃圾回收类型 (`FG_GC` 或 `BG_GC`)。
    *   `unsigned int segno`:  源数据块所在的 segment 号。
    *   `int off`:  源数据块在 segment 内的偏移量。

*   **获取 Address Space Mapping 和 `f2fs_io_info` 初始化:**  与 `ra_data_block` 类似，获取 Address Space Mapping，并初始化 `f2fs_io_info` 结构体 `fio`。  注意，`fio.temp` 设置为 `COLD`，`fio.op` 设置为 `REQ_OP_READ` (初始操作为读)。

*   **获取数据页面缓存页 (不读取):**  `page = f2fs_grab_cache_page(mapping, bidx, false);`:  调用 `f2fs_grab_cache_page` 函数 **获取数据页面缓存页，但不进行读取操作 (`false` 参数)**。  **`move_data_block` 函数不直接操作数据页面的内容，而是通过元数据 mapping 的页面缓存来间接操作数据块。**

*   **块有效性检查和 GC Pinned Control:**  与 `gc_data_segment` Phase 4 类似，进行块有效性检查和 GC Pinned Control 检查。

*   **Dnode 查找:**  通过 `f2fs_get_dnode_of_data` 函数查找数据块地址。

*   **`NULL_ADDR` 检查:**  检查数据块地址是否为 `NULL_ADDR`，如果是，则清除页面 uptodate 标志，并返回 `-ENOENT` 错误。

*   **等待页面和块设备 writeback 完成:**  与 `ra_data_block` 类似，等待数据页面和块设备 writeback 完成。

*   **获取节点信息:**  `err = f2fs_get_node_info(fio.sbi, dn.nid, &ni, false);`:  获取节点信息。

*   **设置 `fio.page` 和 `fio.new_blkaddr`:**  将数据页面 `page` 和数据块地址 `dn.data_blkaddr` 赋值给 `fio` 结构体。

*   **IO 顺序锁获取 (LFS 模式):**  `if (lfs_mode) f2fs_down_write(&fio.sbi->io_order_lock);`:  如果文件系统处于 LFS (Log-structured File System) 模式，则获取 `io_order_lock` 写锁。  **LFS 模式可能需要更严格的 IO 顺序控制。**

*   **获取元数据 mapping 页面缓存页:**  `mpage = f2fs_grab_cache_page(META_MAPPING(fio.sbi), fio.old_blkaddr, false);`:  调用 `f2fs_grab_cache_page` 函数 **从元数据 mapping 的页面缓存中获取页面缓存页 `mpage`，索引为 *旧块地址* (`fio.old_blkaddr`)**。  **`mpage` 将用于缓存 *源数据块* 的内容。**

*   **设置 `fio.encrypted_page`:**  `fio.encrypted_page = mpage;`:  将 `mpage` 赋值给 `fio.encrypted_page`，表示加密页面使用元数据 mapping 页面。

*   **读取源数据块到元数据 mapping 页面缓存:**
    ```c
    /* read source block in mpage */
    if (!PageUptodate(mpage)) {
        err = f2fs_submit_page_bio(&fio);
        if (err) {
            f2fs_put_page(mpage, 1);
            goto up_out;
        }

        f2fs_update_iostat(fio.sbi, inode, FS_DATA_READ_IO,
                            F2FS_BLKSIZE);
        f2fs_update_iostat(fio.sbi, NULL, FS_GDATA_READ_IO,
                            F2FS_BLKSIZE);

        lock_page(mpage);
        if (unlikely(mpage->mapping != META_MAPPING(fio.sbi) ||
                        !PageUptodate(mpage))) {
            err = -EIO;
            f2fs_put_page(mpage, 1);
            goto up_out;
        }
    }
    ```
    *   **检查 `mpage` 是否 uptodate:**  `if (!PageUptodate(mpage)) { ... }`:  如果 `mpage` 不是 uptodate，则需要从磁盘读取源数据块。
    *   **提交页面读取 BIO 请求:**  `err = f2fs_submit_page_bio(&fio);`:  调用 `f2fs_submit_page_bio` 函数 **提交页面读取 BIO 请求，将源数据块读取到 `mpage` (元数据 mapping 页面缓存) 中**。
    *   **更新 IO 统计信息:**  更新数据读取 IO 统计信息。
    *   **再次检查 `mpage` 有效性:**  读取完成后，再次检查 `mpage` 的 mapping 和 uptodate 状态，确保页面读取成功且未被错误地添加到其他 mapping。

*   **准备 Summary 信息:**  `set_summary(&sum, dn.nid, dn.ofs_in_node, ni.version);`:  准备 Summary 信息，用于新数据块的分配。

*   **分配新的数据块地址:**  `err = f2fs_allocate_data_block(fio.sbi, NULL, fio.old_blkaddr, &newaddr, &sum, type, NULL);`:  调用 `f2fs_allocate_data_block` 函数 **分配新的数据块地址 `newaddr`**。  `fio.old_blkaddr` 作为参数传递，可能用于指示要替换的旧块地址。

*   **获取新的元数据 mapping 页面缓存页:**  `fio.encrypted_page = f2fs_pagecache_get_page(META_MAPPING(fio.sbi), newaddr, FGP_LOCK | FGP_CREAT, GFP_NOFS);`:  调用 `f2fs_pagecache_get_page` 函数 **从元数据 mapping 的页面缓存中获取页面缓存页 `fio.encrypted_page`，索引为 *新块地址* (`newaddr`)**。  **`fio.encrypted_page` 将用于写入 *目标数据块* 的内容。**

*   **数据复制:**  `memcpy(page_address(fio.encrypted_page), page_address(mpage), PAGE_SIZE);`:  **将源数据块的内容 (从 `mpage` 中读取) *复制* 到目标数据块的页面缓存 `fio.encrypted_page` 中**。  **数据复制操作在内存中完成，没有直接操作原始数据页面 `page`。**

*   **失效内部缓存:**  `f2fs_invalidate_internal_cache(fio.sbi, fio.old_blkaddr);`:  **失效内部缓存**，可能用于清除与旧块地址相关的缓存信息。

*   **标记目标页面为 dirty 和启动 writeback:**
    ```c
    set_page_dirty(fio.encrypted_page);
    if (clear_page_dirty_for_io(fio.encrypted_page))
        dec_page_count(fio.sbi, F2FS_DIRTY_META);

    set_page_writeback(fio.encrypted_page);
    ```
    *   **标记 `fio.encrypted_page` 为 dirty:**  `set_page_dirty(fio.encrypted_page);`:  将目标页面标记为 dirty，表示页面内容已修改，需要写回磁盘。
    *   **清除 dirty 标志 (如果可以):**  `if (clear_page_dirty_for_io(fio.encrypted_page)) dec_page_count(fio.sbi, F2FS_DIRTY_META);`:  尝试清除目标页面的 dirty 标志，并减少脏元数据页面计数器。  `clear_page_dirty_for_io` 可能会失败，如果页面正忙于 IO 操作。
    *   **启动目标页面的 writeback:**  `set_page_writeback(fio.encrypted_page);`:  启动目标页面的 writeback 流程。

*   **提交页面写 BIO 请求:**
    ```c
    fio.op = REQ_OP_WRITE;
    fio.op_flags = REQ_SYNC;
    fio.new_blkaddr = newaddr;
    f2fs_submit_page_write(&fio);
    ```
    *   **设置 `fio.op` 为 `REQ_OP_WRITE` 和 `fio.op_flags` 为 `REQ_SYNC`:**  将 `fio` 结构体的操作类型设置为写 (`REQ_OP_WRITE`)，操作标志设置为同步写 (`REQ_SYNC`)。
    *   **设置 `fio.new_blkaddr` 为 `newaddr`:**  将 `fio.new_blkaddr` 设置为新分配的块地址 `newaddr`，表示写 IO 的目标地址。
    *   **提交页面写 BIO 请求:**  `f2fs_submit_page_write(&fio);`:  调用 `f2fs_submit_page_write` 函数 **提交页面写 BIO 请求，将目标页面 `fio.encrypted_page` 的内容写入到新块地址 `newaddr`**。  **这是一个同步写操作，会阻塞等待 IO 完成。**

*   **更新 IO 统计信息:**  `f2fs_update_iostat(fio.sbi, NULL, FS_GC_DATA_IO, F2FS_BLKSIZE);`:  更新 GC 数据写 IO 统计信息.

*   **更新数据块地址和设置 Inode 标志:**  `f2fs_update_data_blkaddr(&dn, newaddr); set_inode_flag(inode, FI_APPEND_WRITE);`:  调用 `f2fs_update_data_blkaddr` 函数 **更新 Inode 的元数据，将数据块的地址更新为新分配的地址 `newaddr`**。  `set_inode_flag(inode, FI_APPEND_WRITE);` 设置 Inode 的 `FI_APPEND_WRITE` 标志，可能与 append-write 优化有关。

*   **释放页面引用计数:**  释放目标页面和原始数据页面的引用计数。

*   **`recover_block:` 标签和块替换 (错误处理):**
    ```c
    recover_block:
	if (err)
		f2fs_do_replace_block(fio.sbi, &sum, newaddr, fio.old_blkaddr,
							true, true, true);
    ```
    *   **`recover_block:` 标签:**  在页面分配失败等错误情况下，跳转到 `recover_block` 标签。
    *   **块替换操作:**  `f2fs_do_replace_block(fio.sbi, &sum, newaddr, fio.old_blkaddr, true, true, true);`:  如果发生错误，则调用 `f2fs_do_replace_block` 函数 **执行块替换操作**。  `f2fs_do_replace_block` 函数可能用于在数据迁移失败的情况下，将新分配的块地址替换回旧的块地址，以保证数据一致性。  **这是一种错误恢复机制。**

*   **`up_out:` 标签和 IO 顺序锁释放 (LFS 模式):**  `if (lfs_mode) f2fs_up_write(&fio.sbi->io_order_lock);`:  如果处于 LFS 模式，则释放 `io_order_lock` 写锁。

*   **`put_out:` 标签和 `f2fs_put_dnode` 调用:**  `f2fs_put_dnode(&dn);`:  释放 `dnode_of_data` 结构体 `dn`。

*   **`out:` 标签和页面释放:**  `f2fs_put_page(page, 1);`:  释放原始数据页面 `page` 的引用计数。

*   **返回值:**  函数返回错误码 `err`。

*   **总结 `move_data_block`:**  `move_data_block` 函数实现了 **数据块的块级别迁移**，可能专门用于元数据 Inode GC 场景。  它 **通过元数据 mapping 的页面缓存来间接操作数据块**，将源数据块读取到元数据 mapping 页面缓存，然后分配新的数据块地址，将数据复制到新的数据块地址，并更新 Inode 元数据和 NAT 表。  `move_data_block` 函数的实现非常复杂，包含了 **多重错误处理、并发控制、数据复制、IO 提交、元数据更新和错误恢复** 等一系列操作，保证了数据块迁移的 **可靠性和数据一致性**。  **`move_data_block` 函数与 `move_data_page` 函数相比，可能具有更高的性能开销，但可能更适用于元数据 Inode GC 等特殊场景。**