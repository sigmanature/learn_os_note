**bio结构体和bio迭代器详解**
* [bio迭代器详解](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/block/bio.c/bio迭代器详解.md)
```C
/**
 *	bio_add_page	-	attempt to add page(s) to bio
 *	@bio: destination bio
 *	@page: start page to add
 *	@len: vec entry length, may cross pages
 *	@offset: vec entry offset relative to @page, may cross pages
 *
 *	Attempt to add page(s) to the bio_vec maplist. This will only fail
 *	if either bio->bi_vcnt == bio->bi_max_vecs or it's a cloned bio.
 */
 /*bio_add_folio中的展开形式是bio_add_page(bio, folio_page(folio, off/PAGE_SIZE), len, off % PAGE_SIZE)
 也就是起始页是off整除PAGE_SIZE 然后off信息被通过off % PAGE_SIZE
 传递给了bio_add_page*/
int bio_add_page(struct bio *bio, struct page *page,
		 unsigned int len, unsigned int offset)
{
	bool same_page = false; // 用于标记是否与前一个 bio 向量条目合并的是同一页。

	if (WARN_ON_ONCE(bio_flagged(bio, BIO_CLONED))) // 如果 bio 标记为 BIO_CLONED，发出警告并返回 0 (表示未添加任何页，即失败)。BIO_CLONED 的 bio 不允许添加新的页。
		return 0;
	if (bio->bi_iter.bi_size > UINT_MAX - len) // 检查添加 len 长度后，bio 的总大小是否会溢出 UINT_MAX。如果溢出，返回 0 (失败)。
		return 0;

	if (bio->bi_vcnt > 0 && // 如果 bio 已经有向量条目 (bi_vcnt > 0)。
	    bvec_try_merge_page(&bio->bi_io_vec[bio->bi_vcnt - 1], // 尝试与最后一个向量条目合并。
				page, len, offset, &same_page)) { // 传入最后一个向量条目的地址，要添加的页，长度，偏移量，以及 same_page 指针。
		bio->bi_iter.bi_size += len; // 如果合并成功，增加 bio 的总大小。
		return len; // 返回 len，表示成功添加了 len 长度的数据。
	}

	if (bio->bi_vcnt >= bio->bi_max_vecs) // 如果 bio 的向量条目数量已经达到最大值 (bi_max_vecs)。bi_max_vecs其实就是bio中的bi
		return 0; // 返回 0 (失败)，无法添加更多向量条目。
	__bio_add_page(bio, page, len, offset); // 如果以上条件都不满足，则调用 __bio_add_page 真正添加新的页到 bio。
	return len; // 返回 len，表示成功添加了 len 长度的数据。
}
EXPORT_SYMBOL(bio_add_page); // 导出符号，使其可以在内核模块中使用。
```
```C
/**
 * __bio_add_page - add page(s) to a bio in a new segment
 * @bio: destination bio
 * @page: start page to add
 * @len: length of the data to add, may cross pages
 * @off: offset of the data relative to @page, may cross pages
 *
 * Add the data at @page + @off to @bio as a new bvec.  The caller must ensure
 * that @bio has space for another bvec.
 */
void __bio_add_page(struct bio *bio, struct page *page,
		unsigned int len, unsigned int off)
{
	WARN_ON_ONCE(bio_flagged(bio, BIO_CLONED));
	WARN_ON_ONCE(bio_full(bio, len));

	bvec_set_page(&bio->bi_io_vec[bio->bi_vcnt], page, len, off);/*从bio的bi_vcnt处加页,说明bio本身是维护一个计数的。
	bio的bi_vcnt索引的bvec的起始page被设置为当前这个page,长度被设置为传入的字节数。当然,可以跨page。off代表起始偏移*/
	bio->bi_iter.bi_size += len;/*迭代器要处理的总字节数加上了len长度。*/
	bio->bi_vcnt++;
}
```