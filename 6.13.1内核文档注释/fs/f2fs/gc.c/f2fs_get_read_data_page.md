```c
struct page *f2fs_get_read_data_page(struct inode *inode, pgoff_t index,
				     blk_opf_t op_flags, bool for_write,
				     pgoff_t *next_pgofs)
{
	struct address_space *mapping = inode->i_mapping;
	struct dnode_of_data dn;
	struct page *page;
	int err;

	page = f2fs_grab_cache_page(mapping, index, for_write);
	if (!page)
		return ERR_PTR(-ENOMEM);

	if (f2fs_lookup_read_extent_cache_block(inode, index,
						&dn.data_blkaddr)) {
		if (!f2fs_is_valid_blkaddr(F2FS_I_SB(inode), dn.data_blkaddr,
						DATA_GENERIC_ENHANCE_READ)) {
			err = -EFSCORRUPTED;
			goto put_err;
		}
		goto got_it;
	}

	set_new_dnode(&dn, inode, NULL, NULL, 0);
	err = f2fs_get_dnode_of_data(&dn, index, LOOKUP_NODE);
	if (err) {
		if (err == -ENOENT && next_pgofs)
			*next_pgofs = f2fs_get_next_page_offset(&dn, index);
		goto put_err;
	}
	f2fs_put_dnode(&dn);

	if (unlikely(dn.data_blkaddr == NULL_ADDR)) {
		err = -ENOENT;
		if (next_pgofs)
			*next_pgofs = index + 1;
		goto put_err;
	}
	if (dn.data_blkaddr != NEW_ADDR &&
			!f2fs_is_valid_blkaddr(F2FS_I_SB(inode),
						dn.data_blkaddr,
						DATA_GENERIC_ENHANCE)) {
		err = -EFSCORRUPTED;
		goto put_err;
	}
got_it:
	if (PageUptodate(page)) {
		unlock_page(page);
		return page;
	}

	/*
	 * A new dentry page is allocated but not able to be written, since its
	 * new inode page couldn't be allocated due to -ENOSPC.
	 * In such the case, its blkaddr can be remained as NEW_ADDR.
	 * see, f2fs_add_link -> f2fs_get_new_data_page ->
	 * f2fs_init_inode_metadata.
	 */
	if (dn.data_blkaddr == NEW_ADDR) {
		zero_user_segment(page, 0, PAGE_SIZE);
		if (!PageUptodate(page))
			SetPageUptodate(page);
		unlock_page(page);
		return page;
	}

	err = f2fs_submit_page_read(inode, page_folio(page), dn.data_blkaddr,
						op_flags, for_write);
	if (err)
		goto put_err;
	return page;

put_err:
	f2fs_put_page(page, 1);
	return ERR_PTR(err);
}
```

*   **功能:** `f2fs_get_read_data_page` 函数用于 **获取并读取一个数据页面**。  它会尝试从 extent cache 或 dnode 中查找数据块地址，然后从页面缓存或磁盘读取数据页面，并进行一系列的错误检查和处理。  **`f2fs_get_read_data_page` 函数是 F2FS 中获取数据页面的常用函数。**

*   **参数:**
    *   `struct inode *inode`:  Inode 结构体指针，表示要获取数据页面所属的 Inode。
    *   `pgoff_t index`:  要获取的数据页面在 Inode 地址空间中的块索引。
    *   `blk_opf_t op_flags`:  IO 操作标志，例如 `REQ_RAHEAD` (预读)。
    *   `bool for_write`:  是否用于写操作。  `for_write` 参数可能影响页面缓存的获取方式 (例如，是否需要排他访问)。
    *   `pgoff_t *next_pgofs`:  输出参数，用于返回下一个页面的偏移量 (如果发生 `-ENOENT` 错误)。

*   **获取 Address Space Mapping:**  `struct address_space *mapping = inode->i_mapping;`:  获取 Inode 的 Address Space Mapping。

*   **获取页面缓存页:**  `page = f2fs_grab_cache_page(mapping, index, for_write);`:  调用 `f2fs_grab_cache_page` 函数 **获取数据页面缓存页**。  `for_write` 参数传递给 `f2fs_grab_cache_page`。

*   **Extent Cache 查找:**  与 `ra_data_block` 函数类似，`f2fs_get_read_data_page` 也 **优先尝试从 extent cache 中查找数据块地址**。

*   **Dnode 查找:**  如果 extent cache 未命中，则 **通过 dnode 查找数据块地址**。

*   **`got_it:` 标签和页面处理:**
    ```c
    got_it:
	if (PageUptodate(page)) {
		unlock_page(page);
		return page;
	}

	/*
	 * A new dentry page is allocated but not able to be written, since its
	 * new inode page couldn't be allocated due to -ENOSPC.
	 * In such the case, its blkaddr can be remained as NEW_ADDR.
	 * see, f2fs_add_link -> f2fs_get_new_data_page ->
	 * f2fs_init_inode_metadata.
	 */
	if (dn.data_blkaddr == NEW_ADDR) {
		zero_user_segment(page, 0, PAGE_SIZE);
		if (!PageUptodate(page))
			SetPageUptodate(page);
		unlock_page(page);
		return page;
	}

	err = f2fs_submit_page_read(inode, page_folio(page), dn.data_blkaddr,
						op_flags, for_write);
	if (err)
		goto put_err;
	return page;
    ```
    *   **`PageUptodate` 状态检查:**  `if (PageUptodate(page)) { ... }`:  **检查页面是否已经是 uptodate 状态**。  如果是，则直接解锁页面并返回，避免重复读取。
    *   **`NEW_ADDR` 特殊处理:**
        ```c
        if (dn.data_blkaddr == NEW_ADDR) {
            zero_user_segment(page, 0, PAGE_SIZE);
            if (!PageUptodate(page))
                SetPageUptodate(page);
            unlock_page(page);
            return page;
        }
        ```
        *   **检查数据块地址是否为 `NEW_ADDR`**。  `NEW_ADDR` 表示新分配但尚未写入的块地址。  **注释中解释，这种情况可能发生在创建新的 dentry 页面时，由于空间不足，新的 inode 页面无法分配，导致 dentry 页面的块地址仍然为 `NEW_ADDR`。**
        *   **`NEW_ADDR` 处理:**  如果数据块地址为 `NEW_ADDR`，则 **将页面内容清零 (`zero_user_segment`)，并设置页面为 uptodate 状态 (`SetPageUptodate`)**。  **对于 `NEW_ADDR` 页面，不需要从磁盘读取，直接返回清零后的页面。**
    *   **提交页面读取请求:**  `err = f2fs_submit_page_read(inode, page_folio(page), dn.data_blkaddr, op_flags, for_write);`:  调用 `f2fs_submit_page_read` 函数 **提交页面读取请求**。  `op_flags` 参数从函数参数传递，可以是 `REQ_RAHEAD` 或 `0` (非预读)。

*   **错误处理和返回值:**  函数包含错误处理分支 (`put_err` 标签)，在发生错误时，会释放页面引用计数，并返回错误指针。  函数成功时返回页面指针，错误时返回错误指针。

*   **总结 `f2fs_get_read_data_page`:**  `f2fs_get_read_data_page` 函数是 F2FS 中 **获取和读取数据页面的通用函数**。  它会 **优先从 extent cache 查找数据块地址，然后尝试 dnode 查找**。  函数内部进行了 **页面缓存命中检查、`NEW_ADDR` 特殊处理、页面读取 BIO 提交和错误处理** 等操作，保证了数据页面获取的效率和可靠性。  **`f2fs_get_read_data_page` 函数是 F2FS 数据访问路径上的关键函数。**