**相关函数**
* [f2fs_map_blocks](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_map_blocks.md)
```C
void f2fs_update_read_extent_cache_range(struct dnode_of_data *dn,
				pgoff_t fofs, block_t blkaddr, unsigned int len)
{
	struct extent_info ei = {
		.fofs = fofs,
		.len = len,
		.blk = blkaddr,/*注意extent_info的blkaddr存放的是起始的物理块地址*/
	};

	if (!__may_extent_tree(dn->inode, EX_READ))
		return;

	__update_extent_tree_range(dn->inode, &ei, EX_READ);
}
```