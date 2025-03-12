static inline pgoff_t current_nat_addr(struct f2fs_sb_info *sbi, nid_t start)
{
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	pgoff_t block_off;
	pgoff_t block_addr;

	/*
	 * block_off = segment_off * 512 + off_in_segment
	 * OLD = (segment_off * 512) * 2 + off_in_segment
	 * NEW = 2 * (segment_off * 512 + off_in_segment) - off_in_segment
	 */
	block_off = NAT_BLOCK_OFFSET(start);

	block_addr = (pgoff_t)(nm_i->nat_blkaddr +
		(block_off << 1) -
		(block_off & (BLKS_PER_SEG(sbi) - 1)));

	if (f2fs_test_bit(block_off, nm_i->nat_bitmap))
		block_addr += BLKS_PER_SEG(sbi);

	return block_addr;
}
