 好的，我们来一起深入剖析 `f2fs_read_single_page`, `f2fs_map_blocks`, `f2fs_map_blocks_cached`, 和 `f2fs_get_dnode_of_data` 这四个函数，理解 F2FS 普通 IO 路径下的 **页面读取流程** 和 **块映射机制**，并解答你关于 "普通 IO 视角 Page Index 如何被使用并建立到物理块号的映射" 的疑问。

**1. `f2fs_read_single_page(struct inode *inode, struct folio *folio, unsigned nr_pages, struct f2fs_map_blocks *map, struct bio **bio_ret, sector_t *last_block_in_bio, struct readahead_control *rac)` 函数详解**

```c
static int f2fs_read_single_page(struct inode *inode, struct folio *folio,
					unsigned nr_pages,
					struct f2fs_map_blocks *map,
					struct bio **bio_ret,
					sector_t *last_block_in_bio,
					struct readahead_control *rac)
{
	struct bio *bio = *bio_ret;
	const unsigned int blocksize = F2FS_BLKSIZE;
	// sector_t block_in_file;
	// sector_t last_block;
	// sector_t last_block_in_file;
	// sector_t block_nr;
	// pgoff_t index = folio_index(folio);
	/*volatile仅仅为了调试用*/
	volatile sector_t block_in_file;
	volatile sector_t last_block;
	volatile sector_t last_block_in_file;
	volatile sector_t block_nr;
	volatile pgoff_t index=folio_index(folio);
	int ret = 0;

	block_in_file = (sector_t)index;
	last_block = block_in_file + nr_pages;
	last_block_in_file = F2FS_BYTES_TO_BLK(f2fs_readpage_limit(inode) +
							blocksize - 1);
	if (last_block > last_block_in_file)
		last_block = last_block_in_file;

	/* just zeroing out page which is beyond EOF */
	if (block_in_file >= last_block)
		goto zero_out;
	/*
	 * Map blocks using the previous result first.
	 */
	if ((map->m_flags & F2FS_MAP_MAPPED) &&
			block_in_file > map->m_lblk &&
			block_in_file < (map->m_lblk + map->m_len))
		goto got_it;

	/*
	 * Then do more f2fs_map_blocks() calls until we are
	 * done with this page.
	 */
	map->m_lblk = block_in_file;
	map->m_len = last_block - block_in_file;

	ret = f2fs_map_blocks(inode, map, F2FS_GET_BLOCK_DEFAULT);
	if (ret)
		goto out;
got_it:
	if ((map->m_flags & F2FS_MAP_MAPPED)) {
		block_nr = map->m_pblk + block_in_file - map->m_lblk;
		folio_set_mappedtodisk(folio);

		if (!f2fs_is_valid_blkaddr(F2FS_I_SB(inode), block_nr,
						DATA_GENERIC_ENHANCE_READ)) {
			ret = -EFSCORRUPTED;
			goto out;
		}
	} else {
zero_out:
		folio_zero_segment(folio, 0, folio_size(folio));
		if (f2fs_need_verity(inode, index) &&
		    !fsverity_verify_folio(folio)) {
			ret = -EIO;
			goto out;
		}
		if (!folio_test_uptodate(folio))
			folio_mark_uptodate(folio);
		folio_unlock(folio);
		goto out;
	}

	/*
	 * This page will go to BIO.  Do we need to send this
	 * BIO off first?
	 */
	if (bio && (!page_is_mergeable(F2FS_I_SB(inode), bio,
				       *last_block_in_bio, block_nr) ||
		    !f2fs_crypt_mergeable_bio(bio, inode, index, NULL))) {
submit_and_realloc:
		f2fs_submit_read_bio(F2FS_I_SB(inode), bio, DATA);
		bio = NULL;
	}
	if (bio == NULL) {
		bio = f2fs_grab_read_bio(inode, block_nr, nr_pages,
				f2fs_ra_op_flags(rac), index,
				false);
		if (IS_ERR(bio)) {
			ret = PTR_ERR(bio);
			bio = NULL;
			goto out;
		}
	}

	/*
	 * If the page is under writeback, we need to wait for
	 * its completion to see the correct decrypted data.
	 */
	f2fs_wait_on_block_writeback(inode, block_nr);

	if (!bio_add_folio(bio, folio, blocksize, 0))
		goto submit_and_realloc;

	inc_page_count(F2FS_I_SB(inode), F2FS_RD_DATA);
	f2fs_update_iostat(F2FS_I_SB(inode), NULL, FS_DATA_READ_IO,
							F2FS_BLKSIZE);
	*last_block_in_bio = block_nr;
out:
	*bio_ret = bio;
	return ret;
}
```

*   **功能:** `f2fs_read_single_page` 函数是 F2FS 普通 IO 路径下 **读取 *单个* 数据页面的核心函数**。  **`f2fs_read_single_page` 函数负责将 *逻辑页面索引* 转换为 *物理块地址*，创建 BIO 请求，并将 BIO 请求提交到块设备，最终将数据页面读取到内存。**  **`f2fs_read_single_page` 是 F2FS 数据读取路径上的关键函数。**

*   **参数:**
    *   `struct inode *inode`:  Inode 结构体指针，表示要读取数据页面所属的 Inode。
    *   `struct folio *folio`:  要读取的数据页面的 Folio 结构体。
    *   `unsigned nr_pages`:  要读取的页面数量 (通常为 1，因为是 `read_single_page`)。
    *   `struct f2fs_map_blocks *map`:  `f2fs_map_blocks` 结构体指针，用于 **存储块映射信息** (逻辑块号到物理块号的映射)。
    *   `struct bio **bio_ret`:  指向 `struct bio *` 的指针，用于 **返回 BIO 请求**。  `f2fs_read_single_page` 函数可能会复用或创建新的 BIO 请求。
    *   `sector_t *last_block_in_bio`:  指向 `sector_t` 的指针，用于 **记录 BIO 请求中 *最后一个块的块号***，用于 BIO 请求的合并优化。
    *   `struct readahead_control *rac`:  Read-ahead control 结构体指针，用于 **控制预读行为**。

*   **变量初始化:**  初始化局部变量 `bio`, `blocksize`, `block_in_file`, `last_block`, `last_block_in_file`, `block_nr`, `index`, `ret`。  **`volatile` 关键字修饰的变量 `block_in_file`, `last_block`, `last_block_in_file`, `block_nr`, `index` 仅用于 *调试目的*，在正常代码中不应该使用 `volatile` 修饰局部变量。**

*   **计算块范围:**
    ```c
    block_in_file = (sector_t)index;
    last_block = block_in_file + nr_pages;
    last_block_in_file = F2FS_BYTES_TO_BLK(f2fs_readpage_limit(inode) +
                            blocksize - 1);
    if (last_block > last_block_in_file)
        last_block = last_block_in_file;
    ```
    *   **`block_in_file = (sector_t)index;`:**  将 Folio 的索引 `index` (逻辑页面索引) 转换为 `block_in_file` (文件内块号)。  **`folio_index(folio)` 返回的是 *逻辑页面索引*，在 F2FS 中，逻辑页面索引通常 *直接对应于文件内的逻辑块号*。**
    *   **`last_block = block_in_file + nr_pages;`:**  计算要读取的 *最后一个块的块号*。
    *   **`last_block_in_file = F2FS_BYTES_TO_BLK(f2fs_readpage_limit(inode) + blocksize - 1);`:**  计算文件的 *最后一个有效数据块的块号* (根据 `f2fs_readpage_limit(inode)` 函数获取的文件大小限制)。
    *   **`if (last_block > last_block_in_file) last_block = last_block_in_file;`:**  **限制要读取的块范围 *不超过文件 EOF (End-of-File)*。**

*   **EOF 之外的页面清零:**
    ```c
    /* just zeroing out page which is beyond EOF */
    if (block_in_file >= last_block)
        goto zero_out;
    ```
    *   **检查 `block_in_file` 是否超出 `last_block` (EOF 范围)**。  如果是，表示要读取的页面 **超出文件 EOF**。
    *   **`goto zero_out;`:**  **跳转到 `zero_out` 标签，对页面进行 *清零* 操作，而不是从磁盘读取。**  **对于超出 EOF 范围的页面，F2FS 直接返回零页。**

*   **尝试复用之前的块映射结果 (Extent Cache 优化):**
    ```c
    /*
     * Map blocks using the previous result first.
     */
    if ((map->m_flags & F2FS_MAP_MAPPED) &&
            block_in_file > map->m_lblk &&
            block_in_file < (map->m_lblk + map->m_len))
        goto got_it;
    ```
    *   **检查 `map->m_flags` 是否设置了 `F2FS_MAP_MAPPED` 标志，以及 `block_in_file` 是否在之前的块映射范围内 (`block_in_file > map->m_lblk && block_in_file < (map->m_lblk + map->m_len)`)**。  **`f2fs_map_blocks` 结构体 `map` 可能 *缓存* 了之前的块映射结果 (例如，从 extent cache 获取的 extent 信息)。**
    *   **`goto got_it;`:**  **如果之前的块映射结果仍然有效，则 *直接跳转到 `got_it` 标签，复用之前的块映射结果，避免重复进行块映射操作*。**  **这是一种 *Extent Cache 优化* 策略，可以提高连续读取的性能。**

*   **调用 `f2fs_map_blocks` 函数进行块映射:**
    ```c
    /*
     * Then do more f2fs_map_blocks() calls until we are
     * done with this page.
     */
    map->m_lblk = block_in_file;
    map->m_len = last_block - block_in_file;

    ret = f2fs_map_blocks(inode, map, F2FS_GET_BLOCK_DEFAULT);
    if (ret)
        goto out;
    ```
    *   **更新 `map->m_lblk` 和 `map->m_len`:**  设置 `map->m_lblk` 为 `block_in_file` (逻辑块号)，`map->m_len` 为要映射的块数量。
    *   **调用 `f2fs_map_blocks(inode, map, F2FS_GET_BLOCK_DEFAULT)` 函数进行 *块映射* 操作**。  **`f2fs_map_blocks` 函数负责将 *逻辑块号范围* (`map->m_lblk` 到 `map->m_lblk + map->m_len`) 转换为 *物理块地址范围*，并将映射结果存储在 `map` 结构体中。**  `F2FS_GET_BLOCK_DEFAULT` 标志表示使用默认的块映射模式。
    *   **错误处理:**  如果 `f2fs_map_blocks` 返回错误，则跳转到 `out` 标签，进行错误处理。

*   **`got_it:` 标签和块地址有效性检查:**
    ```c
    got_it:
	if ((map->m_flags & F2FS_MAP_MAPPED)) {
		block_nr = map->m_pblk + block_in_file - map->m_lblk;
		folio_set_mappedtodisk(folio);

		if (!f2fs_is_valid_blkaddr(F2FS_I_SB(inode), block_nr,
						DATA_GENERIC_ENHANCE_READ)) {
			ret = -EFSCORRUPTED;
			goto out;
		}
	} else {
    zero_out:
		folio_zero_segment(folio, 0, folio_size(folio));
		if (f2fs_need_verity(inode, index) &&
		    !fsverity_verify_folio(folio)) {
			ret = -EIO;
			goto out;
		}
		if (!folio_test_uptodate(folio))
			folio_mark_uptodate(folio);
		folio_unlock(folio);
		goto out;
	}
    ```
    *   **`got_it:` 标签:**  Extent Cache 优化路径和 `f2fs_map_blocks` 块映射路径都会到达 `got_it` 标签。
    *   **检查 `map->m_flags` 是否设置了 `F2FS_MAP_MAPPED` 标志**。  `F2FS_MAP_MAPPED` 标志表示块映射成功，`map` 结构体中包含了有效的物理块地址信息。
    *   **计算物理块号:**  `block_nr = map->m_pblk + block_in_file - map->m_lblk;`:  **根据 `map->m_pblk` (起始物理块号), `block_in_file` (当前逻辑块号) 和 `map->m_lblk` (起始逻辑块号)，计算出 *当前逻辑块号 `block_in_file` 对应的 *物理块号 `block_nr`***。  **`map->m_pblk` 存储的是 *连续逻辑块范围的起始物理块号*，需要根据当前逻辑块号相对于起始逻辑块号的偏移量进行计算。**
    *   **`folio_set_mappedtodisk(folio);`:**  **设置 Folio 的 `mappedtodisk` 标志，表示 Folio 已经映射到磁盘上的物理块。**
    *   **物理块地址有效性检查:**  `if (!f2fs_is_valid_blkaddr(F2FS_I_SB(inode), block_nr, DATA_GENERIC_ENHANCE_READ)) { ret = -EFSCORRUPTED; goto out; }`:  **检查计算出的物理块号 `block_nr` 是否有效**。  如果无效，则返回 `-EFSCORRUPTED` 错误。
    *   **`zero_out:` 标签 (EOF 之外的页面清零):**  如果 `map->m_flags` 没有设置 `F2FS_MAP_MAPPED` 标志，则跳转到 `zero_out` 标签，执行页面清零操作 (如前所述)。

*   **BIO 请求合并和提交:**
    ```c
    /*
     * This page will go to BIO.  Do we need to send this
     * BIO off first?
     */
    if (bio && (!page_is_mergeable(F2FS_I_SB(inode), bio,
                   *last_block_in_bio, block_nr) ||
        !f2fs_crypt_mergeable_bio(bio, inode, index, NULL))) {
    submit_and_realloc:
        f2fs_submit_read_bio(F2FS_I_SB(inode), bio, DATA);
        bio = NULL;
    }
    if (bio == NULL) {
        bio = f2fs_grab_read_bio(inode, block_nr, nr_pages,
                f2fs_ra_op_flags(rac), index,
                false);
        if (IS_ERR(bio)) {
            ret = PTR_ERR(bio);
            bio = NULL;
            goto out;
        }
    }
    ```
    *   **BIO 请求合并判断:**  `if (bio && (!page_is_mergeable(F2FS_I_SB(inode), bio, *last_block_in_bio, block_nr) || !f2fs_crypt_mergeable_bio(bio, inode, index, NULL))) { ... }`:  **检查是否可以与 *已有的 BIO 请求 `bio`* 进行 *合并***。  合并条件包括：页面是否可合并 (`page_is_mergeable`)，以及加密是否可合并 (`f2fs_crypt_mergeable_bio`)。  **BIO 请求合并可以减少 BIO 请求的数量，提高 IO 效率。**
    *   **`submit_and_realloc:` 标签和 BIO 提交:**  如果无法合并，则跳转到 `submit_and_realloc` 标签，**提交 *已有的 BIO 请求 `bio`* (`f2fs_submit_read_bio`)，并将 `bio` 设置为 `NULL`，表示需要重新分配新的 BIO 请求。**
    *   **分配新的 BIO 请求:**  `if (bio == NULL) { bio = f2fs_grab_read_bio(inode, block_nr, nr_pages, f2fs_ra_op_flags(rac), index, false); ... }`:  **如果 `bio` 为 `NULL` (没有可复用的 BIO 请求)，则调用 `f2fs_grab_read_bio` 函数 *分配新的 BIO 请求***。  `f2fs_grab_read_bio` 函数会根据物理块号 `block_nr`, 页面数量 `nr_pages`, 预读标志 `f2fs_ra_op_flags(rac)`, 逻辑页面索引 `index` 等信息，创建并初始化新的 BIO 请求。
    *   **错误处理:**  如果 `f2fs_grab_read_bio` 返回错误，则跳转到 `out` 标签，进行错误处理.

*   **等待块设备 Writeback 完成:**  `f2fs_wait_on_block_writeback(inode, block_nr);`:  **等待物理块号 `block_nr` 上的 *块设备 Writeback 操作完成***。  **确保在读取数据之前，任何正在进行的 Writeback 操作都已完成，以避免数据不一致。**

*   **将 Folio 添加到 BIO 请求:**  `if (!bio_add_folio(bio, folio, blocksize, 0)) goto submit_and_realloc;`:  **调用 `bio_add_folio` 函数将 *当前 Folio `folio`* 添加到 *BIO 请求 `bio`* 中**。  `bio_add_folio` 函数负责将 Folio 关联到 BIO 请求，并设置 BIO 请求的相关参数 (例如，数据传输方向，块大小，偏移量等)。  **如果 `bio_add_folio` 返回失败 (例如，BIO 请求已满)，则跳转到 `submit_and_realloc` 标签，提交当前的 BIO 请求，并重新分配新的 BIO 请求。**

*   **更新页面计数和 IO 统计信息:**  `inc_page_count(F2FS_I_SB(inode), F2FS_RD_DATA); f2fs_update_iostat(F2FS_I_SB(inode), NULL, FS_DATA_READ_IO, F2FS_BLKSIZE);`:  **增加数据读取页面计数器 (`F2FS_RD_DATA`)，并更新数据读取 IO 统计信息 (`FS_DATA_READ_IO`)**。

*   **更新 `last_block_in_bio`:**  `*last_block_in_bio = block_nr;`:  **更新 `last_block_in_bio` 指针指向的值，记录当前 BIO 请求中 *最后一个块的块号*，用于后续 BIO 请求的合并判断。**

*   **`out:` 标签和返回值:**  `*bio_ret = bio; return ret;`:  **将 BIO 请求 `bio` 赋值给 `bio_ret` 指针指向的位置，并将错误码 `ret` 返回。**

*   **总结 `f2fs_read_single_page`:**  `f2fs_read_single_page` 函数实现了 **F2FS 普通 IO 路径下读取单个数据页面的完整流程**，包括 **逻辑块号到物理块号的转换 (`f2fs_map_blocks`)、BIO 请求创建和合并、BIO 请求提交、数据读取、错误处理和性能优化 (Extent Cache, BIO 合并, Read-ahead)** 等关键步骤。  **`f2fs_read_single_page` 函数是 F2FS 数据读取路径上的核心函数，体现了 F2FS 在性能和可靠性方面的设计理念。**

**2. `f2fs_map_blocks(struct inode *inode, struct f2fs_map_blocks *map, int flag)` 函数详解**

* [f2fs_map_blocks](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_map_blocks.md)

**3. `f2fs_map_blocks_cached(struct inode *inode, struct f2fs_map_blocks *map, int flag)` 函数详解**

* [f2fs_map_blocks_cached](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_map_blocks_cached.md)

**4. `f2fs_get_dnode_of_data(struct dnode_of_data *dn, pgoff_t index, int mode)` 函数详解**

* [f2fs_get_dnode_of_data](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_get_dnode_of_data.md)
