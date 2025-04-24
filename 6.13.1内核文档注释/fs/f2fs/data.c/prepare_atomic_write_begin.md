专门为使用写时复制 (CoW) 的 F2FS 原子文件处理块查找/预留。
**相关函数**
* [f2fs_write_begin](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_begin.md)
```c
// prepare_atomic_write_begin: 用于原子文件 (CoW) 的 f2fs_write_begin 辅助函数
static int prepare_atomic_write_begin(struct f2fs_sb_info *sbi,
			struct folio *folio, loff_t pos, unsigned int len,
			block_t *blk_addr, bool *node_changed, bool *use_cow)
{
	struct inode *inode = folio->mapping->host; // 原始 inode
	struct inode *cow_inode = F2FS_I(inode)->cow_inode; // 特殊的 CoW inode
	pgoff_t index = folio->index; // 页索引
	/*这个写法很不好,明明可以用位置写的*/
	int err = 0;
	block_t ori_blk_addr = NULL_ADDR; // 原始块地址（如果存在）

	/* 如果 pos 超出文件末尾，则在 COW inode 中预留一个新块 */
	// 如果写入超出原始文件大小，我们肯定需要在 CoW 空间中分配一个新块。
	if ((pos & PAGE_MASK) >= i_size_read(inode))
		goto reserve_block; // 跳过查找，直接进行预留

	/* 首先在 COW inode 中查找块 */
	// 检查此索引的块是否*已经存在*于 CoW inode 的映射中
	// （意味着此页在当前原子操作期间已被写入）。
	err = __find_data_block(cow_inode, index, blk_addr); // 内部块查找函数
	if (err) { // 查找期间出错
		return err;
	} else if (*blk_addr != NULL_ADDR) { // 在 CoW inode 中找到块
		*use_cow = true; // 标记我们正在使用 CoW 映射
		return 0; // 块已存在于 CoW 空间中，可以使用
	}

	// --- 在 CoW inode 中未找到块 ---

	// 如果设置了 FI_ATOMIC_REPLACE 标志，表示我们正在替换整个文件内容。
	// 我们不需要查看原始 inode 的块，只需预留一个新的。
	if (is_inode_flag_set(inode, FI_ATOMIC_REPLACE))
		goto reserve_block;

	/* 在原始 inode 中查找块 */
	// 检查此索引的块是否存在于*原始* inode 的映射中。
	// 如果我们必须在修改之前复制原始数据（CoW），则需要此地址。
	err = __find_data_block(inode, index, &ori_blk_addr);
	if (err) // 在原始 inode 中查找出错
		return err;

reserve_block: // 预留在 CoW inode 中块的公共点
	/* 最后，我们应该在 COW inode 中为更新预留一个新块 */
	// 在 CoW inode 的映射中为此索引预留空间/元数据。
	// 这可能只是更新元数据指针或在日志中预留空间。
	// 如果稍后需要分配新的物理块，'__reserve_data_block' 可能会在 *blk_addr 中返回 NEW_ADDR，
	// 或者可能返回一个预先保留的地址。
	// 'node_changed' 指示节点块（元数据）是否被修改。
	err = __reserve_data_block(cow_inode, index, blk_addr, node_changed);
	if (err) // 预留块出错
		return err;
	inc_atomic_write_cnt(inode); // 增加正在进行的原子写入计数器

	// 如果我们之前找到了原始块 (ori_blk_addr != NULL_ADDR)，
	// 将输出 blk_addr 设置为该原始地址。这告诉 f2fs_write_begin
	// 它需要先从 ori_blk_addr *读取*原始数据，然后再覆盖
	// 页缓存 folio（对应于*新预留的* CoW 块）。
	// 如果 ori_blk_addr 为 NULL_ADDR（原始是空洞），blk_addr 保持为 NEW_ADDR（或 __reserve_data_block 返回的任何值），
	// 表示无需读取旧数据。
	if (ori_blk_addr != NULL_ADDR)
		*blk_addr = ori_blk_addr; // 发出读取原始数据的信号

	// 注意：即使我们使用 ori_blk_addr 进行读取，实际的*写入*最终
	// 也会写入与在 cow_inode 中进行的预留相关联的块。
	*use_cow = true; // 我们现在肯定在使用 CoW 机制。
	return 0; // 成功
}
```