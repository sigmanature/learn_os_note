```C
bool f2fs_load_compressed_folio(struct f2fs_sb_info *sbi, struct folio *folio,
								block_t blkaddr)
{
	struct folio *cfolio;
	bool hitted = false;

	if (!test_opt(sbi, COMPRESS_CACHE))
		return false;

	cfolio = f2fs_filemap_get_folio(COMPRESS_MAPPING(sbi),
				blkaddr, FGP_LOCK | FGP_NOWAIT, GFP_NOFS);/*很奇怪,为什么要以blkaddr为索引去取folio啊?*/
	if (!IS_ERR(cfolio)) {
		if (folio_test_uptodate(cfolio)) {
			atomic_inc(&sbi->compress_page_hit);
			memcpy(folio_address(folio),
				folio_address(cfolio), folio_size(folio));
			hitted = true;
		}
		f2fs_folio_put(cfolio, true);
	}

	return hitted;
}
```