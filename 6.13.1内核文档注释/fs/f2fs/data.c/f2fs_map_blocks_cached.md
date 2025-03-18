 ```c
static bool f2fs_map_blocks_cached(struct inode *inode,
		struct f2fs_map_blocks *map, int flag)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	unsigned int maxblocks = map->m_len;
	pgoff_t pgoff = (pgoff_t)map->m_lblk;
	struct extent_info ei = {};

	if (!f2fs_lookup_read_extent_cache(inode, pgoff, &ei))
		return false;

	map->m_pblk = ei.blk + pgoff - ei.fofs;
	map->m_len = min((pgoff_t)maxblocks, ei.fofs + ei.len - pgoff);
	map->m_flags = F2FS_MAP_MAPPED;
	if (map->m_next_extent)
		*map->m_next_extent = pgoff + map->m_len;

	/* for hardware encryption, but to avoid potential issue in future */
	if (flag == F2FS_GET_BLOCK_DIO)
		f2fs_wait_on_block_writeback_range(inode,
					map->m_pblk, map->m_len);

	if (f2fs_allow_multi_device_dio(sbi, flag)) {
		int bidx = f2fs_target_device_index(sbi, map->m_pblk);
		struct f2fs_dev_info *dev = &sbi->devs[bidx];

		map->m_bdev = dev->bdev;
		map->m_pblk -= dev->start_blk;
		map->m_len = min(map->m_len, dev->end_blk + 1 - map->m_pblk);
	} else {
		map->m_bdev = inode->i_sb->s_bdev;
	}
	return true;
}
```

*   **功能:** `f2fs_map_blocks_cached` 函数是 `f2fs_map_blocks` 函数的 **Extent Cache 快速路径**。  **`f2fs_map_blocks_cached` 函数尝试在 Extent Cache 中 *查找* 逻辑块号 `map->m_lblk` 对应的 *物理块地址范围*，并将查找结果填充到 `f2fs_map_blocks` 结构体 `map` 中。**  **如果 Extent Cache 命中，则可以 *避免* 昂贵的 Dnode 树查找操作，提高块映射效率。**

*   **参数:**
    *   `struct inode *inode`:  Inode 结构体指针，表示要查找 Extent Cache 的文件所属的 Inode。
    *   `struct f2fs_map_blocks *map`:  `f2fs_map_blocks` 结构体指针，用于 **输入逻辑块号** 和 **输出物理块地址范围** 等块映射信息。
    *   `int flag`:  块映射标志，用于 **控制块映射的行为** (虽然在 `f2fs_map_blocks_cached` 中，`flag` 参数的使用相对有限)。

*   **变量初始化:**  初始化局部变量 `sbi`, `maxblocks`, `pgoff`, `ei`。  `ei` 是 `struct extent_info` 类型的变量，用于存储从 Extent Cache 中查找到的 extent 信息。

*   **Extent Cache 查找:**
    ```c
    if (!f2fs_lookup_read_extent_cache(inode, pgoff, &ei))
        return false;
    ```
    *   **调用 `f2fs_lookup_read_extent_cache(inode, pgoff, &ei)` 函数在 Extent Cache 中查找 *逻辑块号 `pgoff`* 对应的 *extent 信息*，并将查找结果存储在 `ei` 结构体中**。  **`f2fs_lookup_read_extent_cache` 函数是 Extent Cache 查找的核心函数，负责在 Extent Cache 的 B-tree 结构中查找 extent 信息。**
    *   **返回值检查:**  **如果 `f2fs_lookup_read_extent_cache` 函数返回 `false` (表示 Extent Cache 未命中)，则 `f2fs_map_blocks_cached` 函数也返回 `false`，表示 Extent Cache 查找失败。**

*   **填充 `f2fs_map_blocks` 结构体:**
    ```c
    map->m_pblk = ei.blk + pgoff - ei.fofs;
    map->m_len = min((pgoff_t)maxblocks, ei.fofs + ei.len - pgoff);
    map->m_flags = F2FS_MAP_MAPPED;
    if (map->m_next_extent)
        *map->m_next_extent = pgoff + map->m_len;
    ```
    *   **计算起始物理块地址:**  `map->m_pblk = ei.blk + pgoff - ei.fofs;`:  **根据 Extent Cache 返回的 extent 信息 `ei`，计算 *起始物理块地址 `map->m_pblk`***。  `ei.blk` 是 extent 的起始物理块地址，`ei.fofs` 是 extent 的起始逻辑块号，`pgoff` 是要查找的逻辑块号。  **`map->m_pblk` 的计算公式将 extent 的起始物理块地址 `ei.blk` 调整为 *当前逻辑块号 `pgoff` 对应的物理块地址*。**
    *   **计算映射长度:**  `map->m_len = min((pgoff_t)maxblocks, ei.fofs + ei.len - pgoff);`:  **计算 *映射长度 `map->m_len`***。  `maxblocks` 是要映射的最大块数量，`ei.fofs + ei.len - pgoff` 是 extent 中 *剩余可映射的块数量* (从当前逻辑块号 `pgoff` 开始到 extent 结束)。  **`map->m_len` 取两者之间的 *最小值*，确保映射长度不超过请求的最大块数量，也不超过 extent 的剩余长度。**
    *   **设置 `F2FS_MAP_MAPPED` 标志:**  `map->m_flags = F2FS_MAP_MAPPED;`:  **设置 `map->m_flags` 标志为 `F2FS_MAP_MAPPED`，表示块映射成功 (从 Extent Cache 命中)。**
    *   **更新 `map->m_next_extent` (如果需要):**  `if (map->m_next_extent) *map->m_next_extent = pgoff + map->m_len;`:  **如果 `map->m_next_extent` 指针不为空，则 *更新 `map->m_next_extent` 指针指向的值，记录 *下一个 extent 的起始逻辑块号*。**  **`map->m_next_extent` 可能用于预读优化，指示下一个可能需要预读的 extent 的起始位置。**

*   **DIO Writeback 等待 (Direct IO):**
    ```c
    /* for hardware encryption, but to avoid potential issue in future */
    if (flag == F2FS_GET_BLOCK_DIO)
        f2fs_wait_on_block_writeback_range(inode,
                    map->m_pblk, map->m_len);
    ```
    *   **检查是否为 `F2FS_GET_BLOCK_DIO` Direct IO 模式**。
    *   **如果是 Direct IO，则 *等待物理块范围的 Writeback 操作完成* (`f2fs_wait_on_block_writeback_range`)**。  **确保 Direct IO 的数据一致性，即使从 Extent Cache 命中了块映射信息，也需要等待 Writeback 完成。**

*   **多设备 Direct IO 处理:**
    ```c
    if (f2fs_allow_multi_device_dio(sbi, flag)) {
        int bidx = f2fs_target_device_index(sbi, map->m_pblk);
        struct f2fs_dev_info *dev = &sbi->devs[bidx];

        map->m_bdev = dev->bdev;
        map->m_pblk -= dev->start_blk;
        map->m_len = min(map->m_len, dev->end_blk + 1 - map->m_pblk);
    } else {
        map->m_bdev = inode->i_sb->s_bdev;
    }
    ```
    *   **检查是否允许多设备 Direct IO (`f2fs_allow_multi_device_dio`)**。
    *   **如果是多设备 Direct IO，则进行多设备相关的处理**，例如：
        *   **获取目标设备索引 `bidx` (`f2fs_target_device_index`)**。
        *   **获取目标设备信息 `dev` (`sbi->devs[bidx]`)**。
        *   **设置 `map->m_bdev` 为目标设备的块设备 `dev->bdev`**。
        *   **调整 `map->m_pblk` 为相对于目标设备起始块号的偏移量 (`map->m_pblk -= dev->start_blk`)**。
        *   **限制 `map->m_len` 不超过目标设备的剩余空间 (`min(map->m_len, dev->end_blk + 1 - map->m_pblk)`)**。
    *   **如果不是多设备 Direct IO，则设置 `map->m_bdev` 为 Inode 所在文件系统的块设备 `inode->i_sb->s_bdev`**。

*   **返回值:**  `return true;`:  **返回 `true`，表示 Extent Cache 查找成功，块映射信息已填充到 `map` 结构体中。**

*   **总结 `f2fs_map_blocks_cached`:**  `f2fs_map_blocks_cached` 函数实现了 **Extent Cache 快速路径的块映射逻辑**。  **`f2fs_map_blocks_cached` 函数通过在 Extent Cache 中查找 extent 信息，快速获取逻辑块号对应的物理块地址范围，并将结果填充到 `f2fs_map_blocks` 结构体中，从而 *加速块映射过程，提高 IO 性能*。**  **`f2fs_map_blocks_cached` 函数是 F2FS Extent Cache 机制的关键组成部分。**



