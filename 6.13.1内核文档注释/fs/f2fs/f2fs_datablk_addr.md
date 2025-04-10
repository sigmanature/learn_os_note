```
static inline block_t f2fs_data_blkaddr(struct dnode_of_data *dn)
{
	return data_blkaddr(dn->inode, dn->node_page, dn->ofs_in_node);
}
static inline block_t data_blkaddr(struct inode *inode,
			struct folio *node_folio, unsigned int offset)
{
	return le32_to_cpu(*(get_dnode_addr(inode, node_folio) + offset));
}
static inline __le32 *get_dnode_addr(struct inode *inode,
					struct folio *node_folio)
{
	return blkaddr_in_node(F2FS_NODE(&node_folio->page)) +
			get_dnode_base(inode, &node_folio->page);/*先拿到传进来的node自己存的数组基地址。可能是inode,也可能是直接node
*/
}
static inline unsigned int get_dnode_base(struct inode *inode,
					struct page *node_page)
{
	if (!IS_INODE(node_page))
		return 0;

	return inode ? get_extra_isize(inode) :
			offset_in_addr(&F2FS_NODE(node_page)->i);
}
static inline __le32 *blkaddr_in_node(struct f2fs_node *node)
{
	return RAW_IS_INODE(node) ? node->i.i_addr : node->dn.addr;
/*这个应该是拿了一个基地址。因为我们知道f2fs_node中的dn.addr就是直接指针数组。
看这个计算是不允许有间接指针。*/
}
```
