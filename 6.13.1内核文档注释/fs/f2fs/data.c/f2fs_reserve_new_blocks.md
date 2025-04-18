```C
/* dn->ofs_in_node will be returned with up-to-date last block pointer */
int f2fs_reserve_new_blocks(struct dnode_of_data *dn, blkcnt_t count)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(dn->inode);
	int err;

	if (!count)
		return 0;

	if (unlikely(is_inode_flag_set(dn->inode, FI_NO_ALLOC)))
		return -EPERM;
	err = inc_valid_block_count(sbi, dn->inode, &count, true);
	if (unlikely(err))
		return err;

	trace_f2fs_reserve_new_blocks(dn->inode, dn->nid,
						dn->ofs_in_node, count);

	f2fs_wait_on_page_writeback(dn->node_page, NODE, true, true);

	for (; count > 0; dn->ofs_in_node++) {
		block_t blkaddr = f2fs_data_blkaddr(dn);

		if (blkaddr == NULL_ADDR) {
			__set_data_blkaddr(dn, NEW_ADDR);
			count--;
		}
	}

	if (set_page_dirty(dn->node_page))
		dn->node_changed = true;/*想想什么时候page会被设置为dirty啊? 能想到的一个当然是gc。另外f2fs_outplace_write
        中也会进行这样的检查*/
	return 0;
}
```
