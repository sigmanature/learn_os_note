
这个函数及其关键。因为它会直接更改folio的私有数据。
```C
static void f2fs_set_compressed_page(struct page *page,
		struct inode *inode, pgoff_t index, void *data)
{
	struct folio *folio = page_folio(page);

	folio_attach_private(folio, (void *)data);

	/* i_crypto_info and iv index */
	folio->index = index;
	folio->mapping = inode->i_mapping;
}
```