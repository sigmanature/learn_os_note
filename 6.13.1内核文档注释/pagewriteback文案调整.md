## 功能背景
保留压缩簇的那段文案。
## 重大问题分析
保留原来的跨簇了 重试了 解锁了等重大问题以外啊,
我们新加一些决赛期间发现的重大问题。注意我提到的这些改动都只做增量的修改。
1.因为page writeback需要处理多种情况。除了直接由fsync调用将所有脏页写回以外,
还有内核的pflush线程在后台会进行定时的脏页写回。后者触发的时候 会遍历相当数量的已经被写回过的清除脏标记的folio。
此时它们并不会正常走
find dirty range和提交io的逻辑 因此 我们必须手动将这些folio 不论它们的阶数是0还是不是0 都将它们给解锁 并且由于它们根本不会被提交到bio 也自然不会被通过bio的回调调用folio_end_writeback
因此对于这种情况的话我们必须手动在最后将它们给end_writeback。
2.另一大问题是f2fs_do_write_data_page中,这里要强调出是原先内核上游补丁考虑欠妥的地方,它直接将folio->index也就是头页作为块查询的索引了,而我们此时期望的是一个可能是某个子页的索引
3.下一个重大问题是 f2fs_submit_page_bio也是因为内核的改动 导致其将folio给添加到bio的时候,我们想添加的是某个子page,但是这行代码错误地直接将整个folio_size给添加进去了,导致bio中记录的信息
是错误的。所以我们的改动 还是加上了具体我们要将folio的哪个子部分给添加到bio之中。
## 解决方案
针对问题1.我们新增了代码:
```C
static int f2fs_write_cache_folios(struct address_space *mapping,
				   struct writeback_control *wbc,
				   enum iostat_type io_type)
else if(folio_test_locked(folio)){
			// not staged,if the folio didn't be locked, unlock it.
			folio_unlock(folio);
		}
		if(!submitted)
		{
			#ifdef CONFIG_F2FS_DEBUG_PRINT
			f2fs_err(F2FS_I_SB(inode),"write_cache_folios submitted %d :no submitted folio",submitted);
			#endif
			// none of the folio's part go to writeback,we manually end_writeback after unlock it
			if(folio_test_writeback(folio))
				folio_end_writeback(folio);
		}
```
首先我们判断当前的folio是否已经被解锁,如果没被解锁,那么要么是正常走完了find_dirty_range循环 要么是没有正常提交的folio没被解锁。
然后,我们根据返回的脏页数量统计,发现如果说没有提交的话,那么我们手动在这里调用folio_end_writeback
并且我发现,f2fs原生的代码中,在进行原地页面写回的那条路上,之前并没有进行提交数量的统计。因此我将这个逻辑给补上。
```C
int f2fs_submit_page_bio(struct f2fs_io_info *fio)
{
	if (is_read_io(bio_op(bio)))
		f2fs_submit_read_bio(fio->sbi, bio, fio->type);
	else
		f2fs_submit_write_bio(fio->sbi, bio, fio->type);
	fio->submitted = true;//请标红
	return 0;
}
```
针对问题2,我们的改动是
```C
int f2fs_do_write_data_page(struct f2fs_io_info *fio)
if (need_inplace_update(fio) &&
	    f2fs_lookup_read_extent_cache_block(inode, folio->index+folio_page_idx(folio,fio->page),
						&fio->old_blkaddr)) {
		if (!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr,
						DATA_GENERIC_ENHANCE))
			return -EFSCORRUPTED;
		
		ipu_force = true;
		fio->need_lock = LOCK_DONE;
		goto got_it;
	}

	/* Deadlock due to between page->lock and f2fs_lock_op */
	if (fio->need_lock == LOCK_REQ && !f2fs_trylock_op(fio->sbi))
		return -EAGAIN;

	err = f2fs_get_dnode_of_data(&dn, folio->index+folio_page_idx(folio,fio->page), LOOKUP_NODE);
	if (err)
		goto out;
	fio->old_blkaddr = dn.data_blkaddr;
```
我们使用了folio->index+folio_page_idx,来计算出当前的子页面对应的真实的页面索引。
针对问题3,我们的改动是:
```C
int f2fs_submit_page_bio(struct f2fs_io_info *fio)
{
	struct bio *bio;
	struct folio *fio_folio = page_folio(fio->page);
	struct folio *data_folio = fio->encrypted_page ?
			page_folio(fio->encrypted_page) : fio_folio;
	if (!f2fs_is_valid_blkaddr(fio->sbi, fio->new_blkaddr,
			fio->is_por ? META_POR : (__is_meta_io(fio) ?
			META_GENERIC : DATA_GENERIC_ENHANCE)))
		return -EFSCORRUPTED;

	trace_f2fs_submit_folio_bio(data_folio, fio);

	/* Allocate a new bio */
	bio = __bio_alloc(fio, 1);

	f2fs_set_bio_crypt_ctx(bio, fio_folio->mapping->host,
			fio_folio->index, fio, GFP_NOIO);
	bio_add_folio_nofail(bio, data_folio, PAGE_SIZE, folio_page_idx(data_folio,fio->encrypted_page ?fio->encrypted_page:fio->page)<<PAGE_SHIFT);//请标红
必要的时候 请使用tabular来进行跨行。
```
