**相关函数**
* [get_dnode_addr](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/f2fs.h/f2fs_datablk_addr.md)
```C
static inline void *inline_data_addr(struct inode *inode, struct page *page)
{
	__le32 *addr = get_dnode_addr(inode, page);

	return (void *)(addr + DEF_INLINE_RESERVED_SIZE);/*拿到inode中直接指针的基地址 然后再加上预留的大小
    这个地址值实质上已经是真实内存区域起始的inline值了。*/
}
```
这里我们回顾一下`get_dnode_addr`的定义:
```C
static inline __le32 *get_dnode_addr(struct inode *inode,
					struct folio *node_folio)
{
	return blkaddr_in_node(F2FS_NODE(&node_folio->page)) +
			get_dnode_base(inode, &node_folio->page);
}
/*先拿到传进来的node自己存的数组基地址。
可能是inode,也可能是直接node。如果是node的话,那blkaddr_in_node已经返回了node->dn.addr 已经是正规的基地址了。
所以get_node_base就返回了0。如果是inode的话,那就得考虑extra_isize的影响。在i_addr的基地址上还得加上extra_isize。
*/
```