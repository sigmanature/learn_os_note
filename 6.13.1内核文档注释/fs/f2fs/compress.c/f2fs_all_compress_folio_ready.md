```C
bool f2fs_all_cluster_page_ready(struct compress_ctx *cc, struct page **pages,
				int index, int nr_pages, bool uptodate)
{
	// pgidx is the absolute file page index of the first page (pages[index])
	// being considered in the current batch from filemap_get_folios_tag.
	unsigned long pgidx = page_folio(pages[index])->index;
	int i = uptodate ? 0 : 1; // If checking uptodate, start loop from 0, else from 1 (why 1?)

	/*
	 * when uptodate set to true, try to check all pages in cluster is
	 * uptodate or not.
	 */
	// (A) If we are checking for uptodate, and the starting page 'pgidx'
	// is not aligned to the start of a cluster, then it's impossible for
	// all pages of *its* cluster to be ready starting from 'pgidx'.
	// This check ensures that if we are to verify a whole cluster starting
	// from pages[index], then pages[index] must be the first page of that cluster.
	if (uptodate && (pgidx % cc->cluster_size))
		return false;

	// (B) Do we have enough pages in the current batch ('pages' array, from 'index' onwards)
	// to cover an entire cluster?
	if (nr_pages - index < cc->cluster_size)
		return false;

	// (C) Loop through the 'cc->cluster_size' pages that would constitute the cluster
	// starting from 'pages[index]'.
	for (; i < cc->cluster_size; i++) {
		// Get the folio for the (index + i)-th page in the batch.
		struct folio *folio = page_folio(pages[index + i]);

		// (D) Check for contiguity: Does this page actually have the expected
		// absolute file page index (pgidx + i)? If not, the pages in the
		// 'pages' array are not contiguous in the file, so they can't form a full cluster.
		if (folio->index != pgidx + i) // This assumes folio is single-page.
			return false;

		// (E) If checking uptodate, is this folio (page) actually uptodate?
		if (uptodate && !folio_test_uptodate(folio))
			return false;
	}

	return true;
}
```