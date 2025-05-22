```C
/**
 * __set_extent_info - Initialize an extent_info structure.
 * @ei: Pointer to the extent_info structure to fill.
 * @fofs: Starting file offset of the extent.
 * @len: Length of the extent in blocks.
 * @blk: Starting block address of the extent.
 * @keep_clen: Flag specific to compression, true to preserve ei->c_len.
 * @age: Age information (for EX_BLOCK_AGE type).
 * @last_blocks: Last blocks info (for EX_BLOCK_AGE type).
 * @type: Type of extent (EX_READ or EX_BLOCK_AGE).
 */
static void __set_extent_info(struct extent_info *ei,
				unsigned int fofs, unsigned int len,
				block_t blk, bool keep_clen,
				unsigned long age, unsigned long last_blocks,
				enum extent_type type)
{
	ei->fofs = fofs; // File offset start
	ei->len = len;   // Length in blocks

	if (type == EX_READ) { // For read extents
		ei->blk = blk; // Block address start
		// If keep_clen is true (used by compressed extent update),
		// we assume c_len was set separately and leave it untouched.
		if (keep_clen)
			return;
#ifdef CONFIG_F2FS_FS_COMPRESSION
		// If not keep_clen, explicitly reset c_len (for non-compressed or merge results).
		ei->c_len = 0;
#endif
	} else if (type == EX_BLOCK_AGE) { // For block age extents (GC related)
		ei->age = age;
		ei->last_blocks = last_blocks;
	}
}

```