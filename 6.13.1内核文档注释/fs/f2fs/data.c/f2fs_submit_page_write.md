去掉了压缩文件以及加密文件等其他东西相关考虑之后保留的定义。
```C
static bool io_is_mergeable(struct f2fs_sb_info *sbi, struct bio *bio,
					struct f2fs_bio_info *io,
					struct f2fs_io_info *fio,
					block_t last_blkaddr,
					block_t cur_blkaddr)
{
	if (!page_is_mergeable(sbi, bio, last_blkaddr, cur_blkaddr))
		return false;
	return io_type_is_mergeable(io, fio);
}
static bool page_is_mergeable(struct f2fs_sb_info *sbi, struct bio *bio,
				block_t last_blkaddr, block_t cur_blkaddr)
{
	if (unlikely(sbi->max_io_bytes &&
			bio->bi_iter.bi_size >= sbi->max_io_bytes))
		return false;
	if (last_blkaddr + 1 != cur_blkaddr)
		return false;
	return bio->bi_bdev == f2fs_target_device(sbi, cur_blkaddr, NULL);
}

void f2fs_submit_page_write(struct f2fs_io_info *fio)
{
	struct f2fs_sb_info *sbi = fio->sbi;
	enum page_type btype = PAGE_TYPE_OF_BIO(fio->type);
	struct f2fs_bio_info *io = sbi->write_io[btype] + fio->temp;
	struct page *bio_page;
	enum count_type type;

	f2fs_bug_on(sbi, is_read_io(fio->op));

	f2fs_down_write(&io->io_rwsem);
next:
	if (fio->in_list) {
		spin_lock(&io->io_lock);
		if (list_empty(&io->io_list)) {
			spin_unlock(&io->io_lock);
			goto out;
		}
		fio = list_first_entry(&io->io_list,
						struct f2fs_io_info, list);
		list_del(&fio->list);
		spin_unlock(&io->io_lock);
	}

	verify_fio_blkaddr(fio);
	else
		bio_page = fio->page;/*fio的page变成了*/

	/* set submitted = true as a return value */
	fio->submitted = 1;

	type = WB_DATA_TYPE(bio_page, fio->compressed_page);
	inc_page_count(sbi, type);

	if (io->bio &&
	    (!io_is_mergeable(sbi, io->bio, io, fio, io->last_block_in_bio,
			      fio->new_blkaddr) ||
	     !f2fs_crypt_mergeable_bio(io->bio, fio->page->mapping->host,
				page_folio(bio_page)->index, fio)))
		__submit_merged_bio(io);
        /*我们最关心的核心逻辑就在这里。拿到fio的块地址之后,怎么和之前的bio进行合并。io_is_mergeable不光看块地址上能不能合并,
        它之后还会看当前传进去的fio和之前的fio是不是同一种io类型
        这里判断出不能合并的话就进行f2fs的__submit_merged_bio操作*/
alloc_new:
	if (io->bio == NULL) {
		io->bio = __bio_alloc(fio, BIO_MAX_VECS);
		f2fs_set_bio_crypt_ctx(io->bio, fio->page->mapping->host,
				page_folio(bio_page)->index, fio, GFP_NOIO);
		io->fio = *fio;
	}

	if (bio_add_page(io->bio, bio_page, PAGE_SIZE, 0) < PAGE_SIZE) {
		__submit_merged_bio(io);
		goto alloc_new;
	}

	if (fio->io_wbc)
		wbc_account_cgroup_owner(fio->io_wbc, page_folio(fio->page),
					 PAGE_SIZE);

	io->last_block_in_bio = fio->new_blkaddr;

	trace_f2fs_submit_folio_write(page_folio(fio->page), fio);
	if (fio->in_list)
		goto next;
out:
	if (is_sbi_flag_set(sbi, SBI_IS_SHUTDOWN) ||
				!f2fs_is_checkpoint_ready(sbi))
		__submit_merged_bio(io);
	f2fs_up_write(&io->io_rwsem);
}
```