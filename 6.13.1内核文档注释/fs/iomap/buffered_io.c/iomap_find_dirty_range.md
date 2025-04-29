```C
static unsigned iomap_find_dirty_range(struct folio *folio, u64 *range_start,
		u64 range_end)
{
	struct iomap_folio_state *ifs = folio->private;

	if (*range_start >= range_end)/*鲁棒性代码*/
		return 0;

	if (ifs)
		return ifs_find_dirty_range(folio, ifs, range_start, range_end);
	return range_end - *range_start;/*如果没分配ifs的话那就设置全区域为脏了*/
}
static unsigned ifs_find_dirty_range(struct folio *folio,
		struct iomap_folio_state *ifs, u64 *range_start, u64 range_end)
{
	struct inode *inode = folio->mapping->host;
	unsigned start_blk =
		offset_in_folio(folio, *range_start) >> inode->i_blkbits;
	unsigned end_blk = min_not_zero(
		offset_in_folio(folio, range_end) >> inode->i_blkbits,
		i_blocks_per_folio(inode, folio));
	unsigned nblks = 1;

	while (!ifs_block_is_dirty(folio, ifs, start_blk))
		if (++start_blk == end_blk)
			return 0;

	while (start_blk + nblks < end_blk) {
		if (!ifs_block_is_dirty(folio, ifs, start_blk + nblks))
			break;
		nblks++;/*看这个循环的实现啊,是找到一个连续的区间 就停下来呢 也就是后续还会有其他的连续的区间呢*/
	}

	*range_start = folio_pos(folio) + (start_blk << inode->i_blkbits);
	return nblks << inode->i_blkbits;
}

```
类比这个函数,我们也能轻易地写出`f2fs_iomap_find_dirty_range`呢:
```C
static unsigned iomap_find_dirty_range(struct folio *folio, u64 *range_start,
		u64 range_end)
{
 if(!folio->mapping)
 	return 0;
 struct inode* inode=folio->mapping->host;
}
```
