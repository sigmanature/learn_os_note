有点像f2fs_map_blocks
```C
static int __f2fs_cluster_blocks(struct inode *inode, unsigned int cluster_idx,
				enum cluster_check_type type)
{
	struct dnode_of_data dn;
	unsigned int start_idx = cluster_idx <<
				F2FS_I(inode)->i_log_cluster_size;/*这个其实是pgoff_t*/
	int ret;

	set_new_dnode(&dn, inode, NULL, NULL, 0);
	ret = f2fs_get_dnode_of_data(&dn, start_idx, LOOKUP_NODE);
	if (ret) {
		if (ret == -ENOENT)
			ret = 0;
		goto fail;
	}

	if (f2fs_sanity_check_cluster(&dn)) {
		ret = -EFSCORRUPTED;
		goto fail;
	}

	if (dn.data_blkaddr == COMPRESS_ADDR) {
		if (type == CLUSTER_COMPR_BLKS)
			ret = 1 + __f2fs_get_cluster_blocks(inode, &dn);
		else if (type == CLUSTER_IS_COMPR)
			ret = 1;/*只是检验一下是不是压缩的*/
	} else if (type == CLUSTER_RAW_BLKS) {
		ret = __f2fs_get_cluster_blocks(inode, &dn);
	}
fail:
	f2fs_put_dnode(&dn);
	return ret;
}
```