```c
/*
 * Calculate start block index indicating the given node offset.
 * Be careful, caller should give this node offset only indicating direct node
 * blocks. If any node offsets, which point the other types of node blocks such
 * as indirect or double indirect node blocks, are given, it must be a caller's
 * bug.
 */
block_t f2fs_start_bidx_of_node(unsigned int node_ofs, struct inode *inode)
{
	unsigned int indirect_blks = 2 * NIDS_PER_BLOCK + 4;
	unsigned int bidx;

	if (node_ofs == 0)
		return 0;

	if (node_ofs <= 2) {
		bidx = node_ofs - 1;
	} else if (node_ofs <= indirect_blks) {
		int dec = (node_ofs - 4) / (NIDS_PER_BLOCK + 1);

		bidx = node_ofs - 2 - dec;
	} else {
		int dec = (node_ofs - indirect_blks - 3) / (NIDS_PER_BLOCK + 1);

		bidx = node_ofs - 5 - dec;
	}
	return bidx * ADDRS_PER_BLOCK(inode) + ADDRS_PER_INODE(inode);
}
```
我们直接代入值硬计算吧。
1.node_ofs=0,此时根本没bidx的事情直接返回0。
2.node_ofs=1,bidx=0,这个值是ADDRS_PER_INODE(inode)也就是INODE存的直接指针的数量。相当于这个时候在inode的第一个直接node块的位置但是还没进去。跨越了所有inode的直接指针。
3.node_ofs=2,bidx=1,这个值是跨越了一个直接node块的所有nid加上inode里面的直接指针。
4.node_ofs=3,dec=(3-4)/(1018+1)=0 bidx此时还是1。也就是还是只跨越了一个直接node块。
