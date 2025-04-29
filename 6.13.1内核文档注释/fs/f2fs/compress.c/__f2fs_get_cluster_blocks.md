**相关函数**
* [__f2fs_cluster_blocks](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/compress.c/__f2fs_cluster_blocks.md)
这个函数给定一个已经被填满的dn和一个inode里的cluster大小,返回其中所有有效块的数量。
```C
static int __f2fs_get_cluster_blocks(struct inode *inode,
					struct dnode_of_data *dn)
{
	unsigned int cluster_size = F2FS_I(inode)->i_cluster_size;
	int count, i;

	for (i = 0, count = 0; i < cluster_size; i++) {
		block_t blkaddr = data_blkaddr(dn->inode, dn->node_page,
							dn->ofs_in_node + i);

		if (__is_valid_data_blkaddr(blkaddr))
			count++;
	}

	return count;
}
```