```C
static int __find_data_block(struct inode *inode, pgoff_t index,
			     block_t *blk_addr)
{
	struct dnode_of_data dn;
	struct folio *ifolio;
	int err = 0;

	ifolio = f2fs_get_inode_folio(F2FS_I_SB(inode), inode->i_ino);
	if (IS_ERR(ifolio))
		return PTR_ERR(ifolio);

	set_new_dnode(&dn, inode, ifolio, ifolio, 0);

	if (!f2fs_lookup_read_extent_cache_block(inode, index,
						 &dn.data_blkaddr)) {
		/* hole case */
		err = f2fs_get_dnode_of_data(&dn, index, LOOKUP_NODE);
		if (err) {
			dn.data_blkaddr = NULL_ADDR;
			err = 0;
		}
	}
	*blk_addr = dn.data_blkaddr;
	f2fs_put_dnode(&dn);
	return err;
}
```