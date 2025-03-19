```C
#define ADDRS_PER_INODE(inode)	addrs_per_page(inode, true)
static inline unsigned int addrs_per_page(struct inode *inode,
							bool is_inode)
{
	unsigned int addrs = is_inode ? (CUR_ADDRS_PER_INODE(inode) -
			get_inline_xattr_addrs(inode)) : DEF_ADDRS_PER_BLOCK;

	if (f2fs_compressed_file(inode))
		return ALIGN_DOWN(addrs, F2FS_I(inode)->i_cluster_size);
	return addrs;
}
#define DEF_ADDRS_PER_BLOCK	((F2FS_BLKSIZE - sizeof(struct node_footer)) / sizeof(__le32))
/* Address Pointers in an Inode */
#define DEF_ADDRS_PER_INODE	((F2FS_BLKSIZE - OFFSET_OF_END_OF_I_EXT	\
					- SIZE_OF_I_NID	\
					- sizeof(struct node_footer)) / sizeof(__le32)) //是923
#define CUR_ADDRS_PER_INODE(inode)	(DEF_ADDRS_PER_INODE - \
					get_extra_isize(inode))
#define OFFSET_OF_END_OF_I_EXT		360
#define SIZE_OF_I_NID			 20 //         就是5个nid的字节大小           
```
![alt text](image.png)
这是f2fsinode块的布局方式。