**相关函数**
* [bio_add_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/block/bio.c/bio_add_page.md)
```C
/**
 * bio_add_folio - Attempt to add part of a folio to a bio.
 * @bio: BIO to add to.
 * @folio: Folio to add.
 * @len: How many bytes from the folio to add.
 * @off: First byte in this folio to add.
 *
 * Filesystems that use folios can call this function instead of calling
 * bio_add_page() for each page in the folio.  If @off is bigger than
 * PAGE_SIZE, this function can create a bio_vec that starts in a page
 * after the bv_page.  BIOs do not support folios that are 4GiB or larger.
 *
 * Return: Whether the addition was successful.
 */
bool bio_add_folio(struct bio *bio, struct folio *folio, size_t len,size_t off)
{
	unsigned long nr = off / PAGE_SIZE;

	if (len > UINT_MAX)
		return false;
	return bio_add_page(bio, folio_page(folio, nr), len, off % PAGE_SIZE) > 0;/*这个函数调用bio_add_page的时候啊,会直接计算给定的off下对应起始哪个page,然后再根据len传入bio_add_page。所以核心是bio_add_page函数怎么转到bio_vec的*/
}
```