```C
/*
 * Structure allocated for each folio to track per-block uptodate, dirty state
 * and I/O completions.
 */
struct iomap_folio_state {
	spinlock_t		state_lock;
	unsigned int		read_bytes_pending;
	atomic_t		write_bytes_pending;

	/*
	 * Each block has two bits in this bitmap:
	 * Bits [0..blocks_per_folio) has the uptodate status.
	 * Bits [b_p_f...(2*b_p_f))   has the dirty status.
	 */
	unsigned long		state[];
};
static struct iomap_folio_state *ifs_alloc(struct inode *inode,
		struct folio *folio, unsigned int flags)
{
	struct iomap_folio_state *ifs = folio->private;
	unsigned int nr_blocks = i_blocks_per_folio(inode, folio);
    /*文件系统块吧?*/
	gfp_t gfp;

	if (ifs || nr_blocks <= 1)/*已经有ifs或者块数量小于1*/
		return ifs;

	if (flags & IOMAP_NOWAIT)
		gfp = GFP_NOWAIT;
	else
		gfp = GFP_NOFS | __GFP_NOFAIL;

	/*
	 * ifs->state tracks two sets of state flags when the
	 * filesystem block size is smaller than the folio size.
	 * The first state tracks per-block uptodate and the
	 * second tracks per-block dirty state.
	 */
	ifs = kzalloc(struct_size(ifs, state,
		      BITS_TO_LONGS(2 * nr_blocks)), gfp);
	if (!ifs)
		return ifs;

	spin_lock_init(&ifs->state_lock);
	if (folio_test_uptodate(folio))
		bitmap_set(ifs->state, 0, nr_blocks);/*从0到nr_blocks-1被设置为1*/
	if (folio_test_dirty(folio))
		bitmap_set(ifs->state, nr_blocks, nr_blocks);/*从nr_blocks到2*nr_blocks-1被设置为1*/
	folio_attach_private(folio, ifs);

	return ifs;
}
```
