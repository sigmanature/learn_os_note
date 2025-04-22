这个板块整理一下所有用到的f2fs从pagecache获取页面中对页面上引用计数的函数。
```mermaid
mindmap
root((__filemap_get_folio))
    pagecache_get_page
        find_get_page
            f2fs_find_data_page
        find_or_create_page-上睡眠锁
            grab_cache_page
                f2fs_grab_cache_page
                    __get_node_page
                        f2fs_get_node_page
```
---
这个板块旨在好好捋顺一下f2fs自己的page产生io的函数和page cache之间进行页面io的并发关系。
首先来看一下page cache之中最基本的等待脏页回写同步原语:
```C
void wait_on_page_writeback(struct page *page)
{
	return folio_wait_writeback(page_folio(page));
}
/**
 * folio_wait_writeback - Wait for a folio to finish writeback.
 * @folio: The folio to wait for.
 *
 * If the folio is currently being written back to storage, wait for the
 * I/O to complete.
 *
 * Context: Sleeps.  Must be called in process context and with
 * no spinlocks held.  Caller should hold a reference on the folio.
 * If the folio is not locked, writeback may start again after writeback
 * has finished.
 */
void folio_wait_writeback(struct folio *folio)
{
	while (folio_test_writeback(folio)) {
		trace_folio_wait_writeback(folio, folio_mapping(folio));
		folio_wait_bit(folio, PG_writeback);
	}
}
/**
 * folio_wait_stable() - wait for writeback to finish, if necessary.
 * @folio: The folio to wait on.
 *
 * This function determines if the given folio is related to a backing
 * device that requires folio contents to be held stable during writeback.
 * If so, then it will wait for any pending writeback to complete.
 *
 * Context: Sleeps.  Must be called in process context and with
 * no spinlocks held.  Caller should hold a reference on the folio.
 * If the folio is not locked, writeback may start again after writeback
 * has finished.
 */
void folio_wait_stable(struct folio *folio)
{
	if (mapping_stable_writes(folio_mapping(folio)))
		folio_wait_writeback(folio);
}
```
`wait_on_page_writeback`是在folio盛行下的兼容函数。会从page获得其头页面然后转成folio。而folio_wait_writeback只要检测到当前的folio设置了PG_writeback,就让当前线程等待。folio_wait_stable就是在folio_wait_writeback前多加了一个判断。主要看当前的mapping有没有设置强迫等待的标志位。
f2fs中除了正常的buffered io 以外,gc也会以块转成单个page为单位进行读和写。
首先对于buffered read来说,不需要考虑这个情况。因为buffered read产生的folio全是新分配的folio,不会产生说正在处于writeback的情况。
然后对于buffered_write,虽然文件系统或者iomap中都会调用__filemap_get_folio,并且f2fs_write_begin中也调用了f2fs_wait_page_writeback函数,但是__filemap_get_folio中的FGP_STABLE标志位一般都不被设置,并且f2fs_wait_page_writeback中走向的是wait_page_stable的分支,而一般address_mapping中不会主动设置强迫稳定的标志位,因此默认情况下buffered_write其实是不会等待脏页写回的。(也比较合理,脏页写回是folio的某个时刻的内容已经提交到bio块设备层了,和folio的内容并无明显关系。buffered write在此期间修改folio的内容完全合理。)
那么需要考虑的应该就是f2fs的gc的读io和写io以及和通用的page writeback之间的同步关系了。
首先我们看一下f2fs中的脏页回写同步原语
```C
void f2fs_folio_wait_writeback(struct folio *folio, enum page_type type,
		bool ordered, bool locked)
{
	if (folio_test_writeback(folio)) {
		struct f2fs_sb_info *sbi = F2FS_F_SB(folio);

		/* submit cached LFS IO */
		f2fs_submit_merged_write_cond(sbi, NULL, &folio->page, 0, type);
		/* submit cached IPU IO */
		f2fs_submit_merged_ipu_write(sbi, NULL, folio);
		if (ordered) {
			folio_wait_writeback(folio);
			f2fs_bug_on(sbi, locked && folio_test_writeback(folio));
		} else {
			folio_wait_stable(folio);
		}
	}
}
void f2fs_wait_on_block_writeback(struct inode *inode, block_t blkaddr)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct folio *cfolio;

	if (!f2fs_meta_inode_gc_required(inode))
		return;

	if (!__is_valid_data_blkaddr(blkaddr))/*跳过无效的块*/
		return;

	cfolio = filemap_lock_folio(META_MAPPING(sbi), blkaddr);
	if (!IS_ERR(cfolio)) {
		f2fs_folio_wait_writeback(cfolio, DATA, true, true);
		f2fs_folio_put(cfolio, true);
	}
}
```
现如今f2fs已经采用了`folio_wait_writeback`。如果要等单个page的话要使用`page_folio(page)`。`wait_on_block_writeback`是用于等元数据(加密文件这种)写回的。gc中,当出现前台gc的时候,`move_data_page`会立刻等待其预读上去的page写回。此时就会出现等待脏页写回的并发问题。
```C
static int move_data_page(struct inode *inode, block_t bidx, int gc_type,
						unsigned int segno, int off)
{
    folio = f2fs_get_lock_data_folio(inode, bidx, true);
    struct f2fs_io_info fio = {
			.ino = inode->i_ino,
			.type = DATA,
			.temp = COLD,
			.old_blkaddr = NULL_ADDR,
			.page = &folio->page,
			.need_lock = LOCK_REQ,
			.io_type = FS_GC_DATA_IO,
		};
        f2fs_folio_wait_writeback(folio, DATA, true, true);
        bool is_dirty = folio_test_dirty(folio);

retry:
		f2fs_folio_wait_writeback(folio, DATA, true, true);

		folio_mark_dirty(folio);
		if (folio_clear_dirty_for_io(folio)) {
			inode_dec_dirty_pages(inode);
			f2fs_remove_dirty_inode(inode);
		}

		set_page_private_gcing(&folio->page);

		err = f2fs_do_write_data_page(&fio);
		if (err) {
			clear_page_private_gcing(&folio->page);
			if (err == -ENOMEM) {
				memalloc_retry_wait(GFP_NOFS);
				goto retry;
			}
			if (is_dirty)
				folio_mark_dirty(folio);
		}
}
```
`move_data_page`是gc中最标准的将页面标记并且回写的函数。这个函数甚至会检查一下之前目标的page是不是已经是脏的了。进行一次回写之后甚至会恢复之前的脏/干净状态。`f2fs_folio_wait_writeback`就像一把锁一样。看正常的脏页回写和gc哪个先抢到锁,哪个就先有脏页回写的权利。并且两者都通过`f2fs_do_write_data_page`函数进行真正的写操作。并且二者也会同时争抢folio锁。
```C

```