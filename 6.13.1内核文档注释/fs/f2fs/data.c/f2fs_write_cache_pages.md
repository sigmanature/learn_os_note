**函数执行流程图**
```mermaid
graph TD
style b fill:#3482,stroke:#333,stroke-width:3
style c fill:#2433,stroke:#333,stroke-width:3
subgraph a[f2fs_write_cache_pages]
0[**初始化总counters<br>nr_pages=0**]-->1
1{脏页数量<=阈值?}--是-->2[inode标记为hot]-->4
1--否-->3[清除hot标记]-->4
4["wbc起始字节->index<br>wbc结束字节->end"]-->5[将范围内所有的脏页全部打上TOWRITE的tag]-->6[index->done_index]-->7[主循环开始]
	
	subgraph b[**主循环**]
	7-->8[根据tag获取nr_folios]-->9[遍历每个folio]-->10[idx=0,p=folio页数]-->c
	subgraph c[**page_loop**]
	11["for idx in range(0,p)"]-->12[添加folio中idx索引的page到栈page数组]-->13[nr_pages++]-->14{nr_pages已经到max_pages?}--是-->15["index更新为[start,end]<br>最后一个folio的起始index,<br>加上当前在folio内部的idx<br>再加1"]-->16[跳转到write循环]
	14--否-->11
	end
	end
end	

```
```C
/*
 * This function was copied from write_cache_pages from mm/page-writeback.c.
 * The major change is making write step of cold data page separately from
 * warm/hot data page.
 */
static int f2fs_write_cache_pages(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	int ret = 0;
	int done = 0, retry = 0;
	/*int done = 0, retry = 0;: 
	声明整型变量 done 和 retry 并初始化为 0。
	done: 可能用作循环控制标志，当设置为非 0 值时，可能表示回写操作完成或需要提前结束。
	retry: 可能用作重试标志，当设置为非 0 值时，可能表示需要重试某些操作。*/
	struct page *pages_local[F2FS_ONSTACK_PAGES];
	/*声明一个 page 结构体指针数组 pages_local，大小为 F2FS_ONSTACK_PAGES。
	这很可能是一个 栈上分配的 局部页数组，用于临时存储要写入的页。 
	F2FS_ONSTACK_PAGES 应该是一个宏定义，表示栈上页数组的最大容量。*/
	struct page **pages = pages_local;
	/*声明一个 page 结构体二级指针 pages，并初始化为指向 pages_local 数组。 
	pages 指针将用于后续操作，这样可以根据需要动态地切换到堆上分配的页数组*/
	struct folio_batch fbatch;
	struct f2fs_sb_info *sbi = F2FS_M_SB(mapping);
	struct bio *bio = NULL;
	sector_t last_block;
#ifdef CONFIG_F2FS_FS_COMPRESSION
	struct inode *inode = mapping->host;
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = NULL_CLUSTER,
		.rpages = NULL,
		.nr_rpages = 0,
		.cpages = NULL,
		.valid_nr_cpages = 0,
		.rbuf = NULL,
		.cbuf = NULL,
		.rlen = PAGE_SIZE * F2FS_I(inode)->i_cluster_size,
		.private = NULL,
	};
#endif
	int nr_folios, p, idx;
	int nr_pages;
	unsigned int max_pages = F2FS_ONSTACK_PAGES;
	pgoff_t index;
	pgoff_t end; /* Inclusive */
	pgoff_t done_index;
	int range_whole = 0;
	xa_mark_t tag;
	int nwritten = 0;
	int submitted = 0;
	int i;

#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (f2fs_compressed_file(inode) &&
	    1 << cc.log_cluster_size > F2FS_ONSTACK_PAGES) {
		pages = f2fs_kzalloc(
			sbi, sizeof(struct page *) << cc.log_cluster_size,
			GFP_NOFS | __GFP_NOFAIL);
		max_pages = 1 << cc.log_cluster_size;/*压缩上下文的情况下,如果一个簇的大小比max_pages大,
		调整这个数目到一个簇的大小上去。*/
	}
#endif

	folio_batch_init(&fbatch);

	if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);

	if (wbc->range_cyclic) {
		/*如果为真，表示回写范围是 循环的 (cyclic)。 
		循环回写通常用于后台周期性的回写操作，从上次停止的位置继续回写。*/
		index = mapping->writeback_index; /* prev offset
		很可能存储了 上次循环回写操作结束时的页偏移量，作为本次循环回写的起始位置。 */
		end = -1;
	} else {
		index = wbc->range_start >> PAGE_SHIFT;
		/*将 wbc->range_start (回写起始位置) 右移 PAGE_SHIFT 位，
		得到起始页的索引，并赋值给 index。*/
		end = wbc->range_end >> PAGE_SHIFT;
		/*将 wbc->range_end (回写结束位置) 右移 PAGE_SHIFT 位，
		得到结束页的索引，并赋值给 end。*/
		if (wbc->range_start == 0 && wbc->range_end == LLONG_MAX)
			range_whole = 1;
		/*条件判断，检查回写范围是否是 整个文件或地址空间。
		如果回写范围是整个文件，则将 range_whole 标志设置为 1。*/
	}
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag = PAGECACHE_TAG_TOWRITE;
	else
		tag = PAGECACHE_TAG_DIRTY;
retry:
	retry = 0;
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag_pages_for_writeback(mapping, index, end);
	/*将范围内所有的脏页全部打上TOWRITE的tag*/
	done_index = index; /*将done_index已经处理完的页初始化为索引起点*/
	while (!done && !retry && (index <= end)) 
   {
		/*循环开始 如果说没有done 没有出现重试并且起始的index小于等于end 等于的情况
		表示一个page*/
		nr_pages = 0;
again:
		nr_folios = filemap_get_folios_tag(mapping, &index, end, tag,
						   &fbatch);
		/*注意这个函数并不上锁*/
		if (nr_folios == 0) {
			if (nr_pages)
				goto write;
			break;
		}

		for (i = 0; i < nr_folios; i++) {
			struct folio *folio = fbatch.folios[i];

			idx = 0;
			p = folio_nr_pages(folio);
add_more:
			pages[nr_pages] = folio_page(folio, idx);
			folio_get(folio);
			if (++nr_pages == max_pages) {
				index = folio->index + idx +
					1; /*要想知道这里folio->index的大小,
				就得知道filemap_get_folios_tag的实现*/
				folio_batch_release(&fbatch);
				goto write;
			}
			if (++idx < p)
				goto add_more;
		} /*实际上代码逻辑可以重写为二重循环:
		for(int i=0;i<nr_folios;++i)
		{
			folio=fbatch.folio[i]
			p=folio_nr_pages(folio);
			int idx=0;
			for(;idx<p;idx++)
			{
				page[nr_pages]=folio_page(folio,idx);
				folio_get(folio);
				if (++nr_pages == max_pages) {
					index = folio->index + idx + 1;
					folio_batch_release(&fbatch);
					goto write;
				}
			}
		}
		*/
		folio_batch_release(&fbatch);
		goto again;
write: /*开始write循环*/
		for (i = 0; i < nr_pages; i++)
		{
			struct page *page = pages[i];
			struct folio *folio = page_folio(page);
			/*看来是把page再转化回folio
			实现上存在疑问 因为这样似乎是之前把所有的folio拆散现在却又单个page转化回去
			*/
			bool need_readd;
readd:
			need_readd = false;
#ifdef CONFIG_F2FS_FS_COMPRESSION
			if (f2fs_compressed_file(inode)) {
				void *fsdata = NULL;
				struct page *pagep;
				int ret2;

				ret = f2fs_init_compress_ctx(&cc);
				if (ret) {
					done = 1;
					break;
				}

				if (!f2fs_cluster_can_merge_page(
					    &cc, folio->index)) {
					ret = f2fs_write_multi_pages(
						&cc, &submitted, wbc, io_type);
					if (!ret)
						need_readd = true;
					goto result;
				}

				if (unlikely(f2fs_cp_error(sbi)))
					goto lock_folio;

				if (!f2fs_cluster_is_empty(&cc))
					goto lock_folio;

				if (f2fs_all_cluster_page_ready(&cc, pages, i,
								nr_pages, true))
					goto lock_folio;

				ret2 = f2fs_prepare_compress_overwrite(
					inode, &pagep, folio->index, &fsdata);/*在脏页回写的时候 也要把当前一整个簇的page全部读到
					page cache里头*/
				if (ret2 < 0) {
					ret = ret2;
					done = 1;
					break;
				} else if (ret2 &&
					   (!f2fs_compress_write_end(
						    inode, fsdata, folio->index,
						    1) ||
					    !f2fs_all_cluster_page_ready(
						    &cc, pages, i, nr_pages,
						    false))) {
					retry = 1;
					break;
				}
			}
#endif
			/* give a priority to WB_SYNC threads */
			if (atomic_read(&sbi->wb_sync_req[DATA]) &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
#ifdef CONFIG_F2FS_FS_COMPRESSION
lock_folio:
#endif
			done_index =
				folio->index; /*更新已完成的index为folio的起始index*/
retry_write:
			/*为了保证并发下的安全性,我们必须在锁的保护下把之前所有检查过的事项全再检查一遍*/
			folio_lock(folio);
			/*
								在锁的保护下检查folio是否还在内存中了
								*/
			if (unlikely(folio->mapping != mapping)) {
continue_unlock:
				folio_unlock(folio); /*不在的话就解锁continue*/
				continue;
			}

			if (!folio_test_dirty(folio)) {
				/* someone wrote it for us */
				goto continue_unlock;
			}

			if (folio_test_writeback(folio)) {
				if (wbc->sync_mode == WB_SYNC_NONE)
					goto continue_unlock;
				f2fs_wait_on_page_writeback(&folio->page, DATA,
							    true, true);
			}

			if (!folio_clear_dirty_for_io(folio))
				goto continue_unlock;

#ifdef CONFIG_F2FS_FS_COMPRESSION
			if (f2fs_compressed_file(inode)) {
				folio_get(folio);
				f2fs_compress_ctx_add_page(&cc, folio);
				continue;
			}
#endif /*还是只写了一页吗?*/
			ret = f2fs_write_single_data_page(folio, &submitted,
							  &bio, &last_block,
							  wbc, io_type, 0,
							  true);
			if (ret == AOP_WRITEPAGE_ACTIVATE)
				folio_unlock(folio);
#ifdef CONFIG_F2FS_FS_COMPRESSION
result:
#endif
			nwritten += submitted;
			wbc->nr_to_write -= submitted;

			if (unlikely(ret)) {
				/*
				 * keep nr_to_write, since vfs uses this to
				 * get # of written pages.
				 */
				if (ret == AOP_WRITEPAGE_ACTIVATE) {
					ret = 0;
					goto next;
				} else if (ret == -EAGAIN) {
					ret = 0;
					if (wbc->sync_mode == WB_SYNC_ALL) {
						f2fs_io_schedule_timeout(
							DEFAULT_IO_TIMEOUT);
						goto retry_write;
					}
					goto next;
				}
				done_index = folio_next_index(folio);
				done = 1;
				break;
			}

			if (wbc->nr_to_write <= 0 &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
next:
			if (need_readd)
				goto readd;
		}
		release_pages(pages, nr_pages);
		cond_resched();
	}
#ifdef CONFIG_F2FS_FS_COMPRESSION
	/* flush remained pages in compress cluster */
	if (f2fs_compressed_file(inode) && !f2fs_cluster_is_empty(&cc)) {
		ret = f2fs_write_multi_pages(&cc, &submitted, wbc, io_type);
		nwritten += submitted;
		wbc->nr_to_write -= submitted;
		if (ret) {
			done = 1;
			retry = 0;
		}
	}
	if (f2fs_compressed_file(inode))
		f2fs_destroy_compress_ctx(&cc, false);
#endif
	if (retry) {
		index = 0;
		end = -1;
		goto retry;
	}
	if (wbc->range_cyclic && !done)
		done_index = 0;
	if (wbc->range_cyclic || (range_whole && wbc->nr_to_write > 0))
		mapping->writeback_index = done_index;

	if (nwritten)
		f2fs_submit_merged_write_cond(F2FS_M_SB(mapping), mapping->host,
					      NULL, 0, DATA);
	/* submit cached bio of IPU write */
	if (bio)
		f2fs_submit_merged_ipu_write(sbi, &bio, NULL);

#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (pages != pages_local)
		kfree(pages);
#endif

	return ret;
}
```
我们可以把所有有关压缩的代码暂时去除来简化一下代码的逻辑,简化之后代码如下:
```C
static int f2fs_write_cache_pages(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	int ret = 0;
	int done = 0, retry = 0;
	/*int done = 0, retry = 0;: 
	声明整型变量 done 和 retry 并初始化为 0。
	done: 可能用作循环控制标志，当设置为非 0 值时，可能表示回写操作完成或需要提前结束。
	retry: 可能用作重试标志，当设置为非 0 值时，可能表示需要重试某些操作。*/
	struct page *pages_local[F2FS_ONSTACK_PAGES];
	/*声明一个 page 结构体指针数组 pages_local，大小为 F2FS_ONSTACK_PAGES。
	这很可能是一个 栈上分配的 局部页数组，用于临时存储要写入的页。 
	F2FS_ONSTACK_PAGES 应该是一个宏定义，表示栈上页数组的最大容量。*/
	struct page **pages = pages_local;
	/*声明一个 page 结构体二级指针 pages，并初始化为指向 pages_local 数组。 
	pages 指针将用于后续操作，这样可以根据需要动态地切换到堆上分配的页数组*/
	struct folio_batch fbatch;
	struct f2fs_sb_info *sbi = F2FS_M_SB(mapping);
	struct bio *bio = NULL;
	sector_t last_block;
	int nr_folios, p, idx;
	int nr_pages;
	unsigned int max_pages = F2FS_ONSTACK_PAGES;
	pgoff_t index;
	pgoff_t end; /* Inclusive */
	pgoff_t done_index;
	int range_whole = 0;
	xa_mark_t tag;
	int nwritten = 0;
	int submitted = 0;
	int i;
	folio_batch_init(&fbatch);

	if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);

	if (wbc->range_cyclic) {
		/*如果为真，表示回写范围是 循环的 (cyclic)。 
		循环回写通常用于后台周期性的回写操作，从上次停止的位置继续回写。*/
		index = mapping->writeback_index; /* prev offset
		很可能存储了 上次循环回写操作结束时的页偏移量，作为本次循环回写的起始位置。 */
		end = -1;
	} else {
		index = wbc->range_start >> PAGE_SHIFT;
		/*将 wbc->range_start (回写起始位置) 右移 PAGE_SHIFT 位，
		得到起始页的索引，并赋值给 index。*/
		end = wbc->range_end >> PAGE_SHIFT;
		/*将 wbc->range_end (回写结束位置) 右移 PAGE_SHIFT 位，
		得到结束页的索引，并赋值给 end。*/
		if (wbc->range_start == 0 && wbc->range_end == LLONG_MAX)
			range_whole = 1;
		/*条件判断，检查回写范围是否是 整个文件或地址空间。
		如果回写范围是整个文件，则将 range_whole 标志设置为 1。*/
	}
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag = PAGECACHE_TAG_TOWRITE;
	else
		tag = PAGECACHE_TAG_DIRTY;
	while (!done && !retry && (index <= end)) 
	{
		/*
		....
		....
		收集nr_pages的循环
		*/
write: /*开始write循环*/
		for (i = 0; i < nr_pages; i++)
		{
			struct page *page = pages[i];
			struct folio *folio = page_folio(page);
			/*看来是把page再转化回folio
			实现上存在疑问 因为这样似乎是之前把所有的folio拆散现在却又单个page转化回去
			*/
			bool need_readd;
readd:
			need_readd = false;
			/* give a priority to WB_SYNC threads */
			if (atomic_read(&sbi->wb_sync_req[DATA]) &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
			done_index =
				folio->index; /*更新已完成的index为folio的起始index*/
retry_write:
			/*为了保证并发下的安全性,我们必须在锁的保护下把之前所有检查过的事项全再检查一遍*/
			folio_lock(folio);
			/*
								在锁的保护下检查folio是否还在内存中了
								*/
			if (unlikely(folio->mapping != mapping)) {
continue_unlock:
				folio_unlock(folio); /*不在的话就解锁continue*/
				continue;
			}

			if (!folio_test_dirty(folio)) {
				/* someone wrote it for us */
				goto continue_unlock;
			}

			if (folio_test_writeback(folio)) {
				if (wbc->sync_mode == WB_SYNC_NONE)
					goto continue_unlock;
				f2fs_wait_on_page_writeback(&folio->page, DATA,
							    true, true);
			}

			if (!folio_clear_dirty_for_io(folio))
				goto continue_unlock;
			ret = f2fs_write_single_data_page(folio, &submitted,
							  &bio, &last_block,
							  wbc, io_type, 0,
							  true);
			if (ret == AOP_WRITEPAGE_ACTIVATE)
				folio_unlock(folio);
#ifdef CONFIG_F2FS_FS_COMPRESSION
result:
#endif
			nwritten += submitted;
			wbc->nr_to_write -= submitted;

			if (unlikely(ret)) {
				/*
				 * keep nr_to_write, since vfs uses this to
				 * get # of written pages.
				 */
				if (ret == AOP_WRITEPAGE_ACTIVATE) {
					ret = 0;
					goto next;
				} else if (ret == -EAGAIN) {
					ret = 0;
					if (wbc->sync_mode == WB_SYNC_ALL) {
						f2fs_io_schedule_timeout(
							DEFAULT_IO_TIMEOUT);
						goto retry_write;
					}
					goto next;
				}
				done_index = folio_next_index(folio);
				done = 1;
				break;
			}

			if (wbc->nr_to_write <= 0 &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
next:
			if (need_readd)
				goto readd;
		}
	}
}
```
**函数执行流程图**
```mermaid
graph TD
style b fill:#3482,stroke:#333,stroke-width:3
style c fill:#2433,stroke:#333,stroke-width:3
subgraph a[f2fs_write_cache_pages]
0[**初始化总counters<br>nr_pages=0**]-->1
1{脏页数量<=阈值?}--是-->2[inode标记为hot]-->4
1--否-->3[清除hot标记]-->4
4["wbc起始字节->index<br>wbc结束字节->end"]-->5[将范围内所有的脏页全部打上TOWRITE的tag]-->6[index->done_index]-->7[主循环开始]
	
	subgraph b[**主循环**]
	7-->8[根据tag获取nr_folios]-->9[遍历每个folio]-->10[idx=0,p=folio页数]-->c
	subgraph c[**page_loop**]
	11["for idx in range(0,p)"]-->12[添加folio中idx索引的page到栈page数组]-->13[nr_pages++]-->14{nr_pages已经到max_pages?}--是-->15["index更新为[start,end]<br>最后一个folio的起始index,<br>加上当前在folio内部的idx<br>再加1"]-->16[跳转到write循环]
	14--否-->11
	end
	end
end	

```
```C
/*
 * This function was copied from write_cache_pages from mm/page-writeback.c.
 * The major change is making write step of cold data page separately from
 * warm/hot data page.
 */
static int f2fs_write_cache_pages(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	int ret = 0;
	int done = 0, retry = 0;
	/*int done = 0, retry = 0;: 
	声明整型变量 done 和 retry 并初始化为 0。
	done: 可能用作循环控制标志，当设置为非 0 值时，可能表示回写操作完成或需要提前结束。
	retry: 可能用作重试标志，当设置为非 0 值时，可能表示需要重试某些操作。*/
	struct page *pages_local[F2FS_ONSTACK_PAGES];
	/*声明一个 page 结构体指针数组 pages_local，大小为 F2FS_ONSTACK_PAGES。
	这很可能是一个 栈上分配的 局部页数组，用于临时存储要写入的页。 
	F2FS_ONSTACK_PAGES 应该是一个宏定义，表示栈上页数组的最大容量。*/
	struct page **pages = pages_local;
	/*声明一个 page 结构体二级指针 pages，并初始化为指向 pages_local 数组。 
	pages 指针将用于后续操作，这样可以根据需要动态地切换到堆上分配的页数组*/
	struct folio_batch fbatch;
	struct f2fs_sb_info *sbi = F2FS_M_SB(mapping);
	struct bio *bio = NULL;
	sector_t last_block;
#ifdef CONFIG_F2FS_FS_COMPRESSION
	struct inode *inode = mapping->host;
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = NULL_CLUSTER,
		.rpages = NULL,
		.nr_rpages = 0,
		.cpages = NULL,
		.valid_nr_cpages = 0,
		.rbuf = NULL,
		.cbuf = NULL,
		.rlen = PAGE_SIZE * F2FS_I(inode)->i_cluster_size,
		.private = NULL,
	};
#endif
	int nr_folios, p, idx;
	int nr_pages;
	unsigned int max_pages = F2FS_ONSTACK_PAGES;
	pgoff_t index;
	pgoff_t end; /* Inclusive */
	pgoff_t done_index;
	int range_whole = 0;
	xa_mark_t tag;
	int nwritten = 0;
	int submitted = 0;
	int i;

#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (f2fs_compressed_file(inode) &&
	    1 << cc.log_cluster_size > F2FS_ONSTACK_PAGES) {
		pages = f2fs_kzalloc(
			sbi, sizeof(struct page *) << cc.log_cluster_size,
			GFP_NOFS | __GFP_NOFAIL);
		max_pages = 1 << cc.log_cluster_size;/*压缩上下文的情况下,如果一个簇的大小比max_pages大,
		调整这个数目到一个簇的大小上去。*/
	}
#endif

	folio_batch_init(&fbatch);

	if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);

	if (wbc->range_cyclic) {
		/*如果为真，表示回写范围是 循环的 (cyclic)。 
		循环回写通常用于后台周期性的回写操作，从上次停止的位置继续回写。*/
		index = mapping->writeback_index; /* prev offset
		很可能存储了 上次循环回写操作结束时的页偏移量，作为本次循环回写的起始位置。 */
		end = -1;
	} else {
		index = wbc->range_start >> PAGE_SHIFT;
		/*将 wbc->range_start (回写起始位置) 右移 PAGE_SHIFT 位，
		得到起始页的索引，并赋值给 index。*/
		end = wbc->range_end >> PAGE_SHIFT;
		/*将 wbc->range_end (回写结束位置) 右移 PAGE_SHIFT 位，
		得到结束页的索引，并赋值给 end。*/
		if (wbc->range_start == 0 && wbc->range_end == LLONG_MAX)
			range_whole = 1;
		/*条件判断，检查回写范围是否是 整个文件或地址空间。
		如果回写范围是整个文件，则将 range_whole 标志设置为 1。*/
	}
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag = PAGECACHE_TAG_TOWRITE;
	else
		tag = PAGECACHE_TAG_DIRTY;
retry:
	retry = 0;
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag_pages_for_writeback(mapping, index, end);
	/*将范围内所有的脏页全部打上TOWRITE的tag*/
	done_index = index; /*将done_index已经处理完的页初始化为索引起点*/
	while (!done && !retry && (index <= end)) 
   {
		/*循环开始 如果说没有done 没有出现重试并且起始的index小于等于end 等于的情况
		表示一个page*/
		nr_pages = 0;
again:
		nr_folios = filemap_get_folios_tag(mapping, &index, end, tag,
						   &fbatch);
		/*注意这个函数并不上锁*/
		if (nr_folios == 0) {
			if (nr_pages)
				goto write;
			break;
		}

		for (i = 0; i < nr_folios; i++) {
			struct folio *folio = fbatch.folios[i];

			idx = 0;
			p = folio_nr_pages(folio);
add_more:
			pages[nr_pages] = folio_page(folio, idx);
			folio_get(folio);
			if (++nr_pages == max_pages) {
				index = folio->index + idx +
					1; /*要想知道这里folio->index的大小,
				就得知道filemap_get_folios_tag的实现*/
				folio_batch_release(&fbatch);
				goto write;
			}
			if (++idx < p)
				goto add_more;
		} /*实际上代码逻辑可以重写为二重循环:
		for(int i=0;i<nr_folios;++i)
		{
			folio=fbatch.folio[i]
			p=folio_nr_pages(folio);
			int idx=0;
			for(;idx<p;idx++)
			{
				page[nr_pages]=folio_page(folio,idx);
				folio_get(folio);
				if (++nr_pages == max_pages) {
					index = folio->index + idx + 1;
					folio_batch_release(&fbatch);
					goto write;
				}
			}
		}
		*/
		folio_batch_release(&fbatch);
		goto again;
write: /*开始write循环*/
		for (i = 0; i < nr_pages; i++)
		{
			struct page *page = pages[i];
			struct folio *folio = page_folio(page);
			/*看来是把page再转化回folio
			实现上存在疑问 因为这样似乎是之前把所有的folio拆散现在却又单个page转化回去
			*/
			bool need_readd;
readd:
			need_readd = false;
#ifdef CONFIG_F2FS_FS_COMPRESSION
			if (f2fs_compressed_file(inode)) {
				void *fsdata = NULL;
				struct page *pagep;
				int ret2;

				ret = f2fs_init_compress_ctx(&cc);
				if (ret) {
					done = 1;
					break;
				}

				if (!f2fs_cluster_can_merge_page(
					    &cc, folio->index)) {
					ret = f2fs_write_multi_pages(
						&cc, &submitted, wbc, io_type);
					if (!ret)
						need_readd = true;
					goto result;
				}

				if (unlikely(f2fs_cp_error(sbi)))
					goto lock_folio;

				if (!f2fs_cluster_is_empty(&cc))
					goto lock_folio;

				if (f2fs_all_cluster_page_ready(&cc, pages, i,
								nr_pages, true))
					goto lock_folio;

				ret2 = f2fs_prepare_compress_overwrite(
					inode, &pagep, folio->index, &fsdata);/*在脏页回写的时候 也要把当前一整个簇的page全部读到
					page cache里头*/
				if (ret2 < 0) {
					ret = ret2;
					done = 1;
					break;
				} else if (ret2 &&
					   (!f2fs_compress_write_end(
						    inode, fsdata, folio->index,
						    1) ||
					    !f2fs_all_cluster_page_ready(
						    &cc, pages, i, nr_pages,
						    false))) {
					retry = 1;
					break;
				}
			}
#endif
			/* give a priority to WB_SYNC threads */
			if (atomic_read(&sbi->wb_sync_req[DATA]) &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
#ifdef CONFIG_F2FS_FS_COMPRESSION
lock_folio:
#endif
			done_index =
				folio->index; /*更新已完成的index为folio的起始index*/
retry_write:
			/*为了保证并发下的安全性,我们必须在锁的保护下把之前所有检查过的事项全再检查一遍*/
			folio_lock(folio);
			/*
								在锁的保护下检查folio是否还在内存中了
								*/
			if (unlikely(folio->mapping != mapping)) {
continue_unlock:
				folio_unlock(folio); /*不在的话就解锁continue*/
				continue;
			}

			if (!folio_test_dirty(folio)) {
				/* someone wrote it for us */
				goto continue_unlock;
			}

			if (folio_test_writeback(folio)) {
				if (wbc->sync_mode == WB_SYNC_NONE)
					goto continue_unlock;
				f2fs_wait_on_page_writeback(&folio->page, DATA,
							    true, true);
			}

			if (!folio_clear_dirty_for_io(folio))
				goto continue_unlock;

#ifdef CONFIG_F2FS_FS_COMPRESSION
			if (f2fs_compressed_file(inode)) {
				folio_get(folio);
				f2fs_compress_ctx_add_page(&cc, folio);
				continue;
			}
#endif /*还是只写了一页吗?*/
			ret = f2fs_write_single_data_page(folio, &submitted,
							  &bio, &last_block,
							  wbc, io_type, 0,
							  true);
			if (ret == AOP_WRITEPAGE_ACTIVATE)
				folio_unlock(folio);
#ifdef CONFIG_F2FS_FS_COMPRESSION
result:
#endif
			nwritten += submitted;
			wbc->nr_to_write -= submitted;

			if (unlikely(ret)) {
				/*
				 * keep nr_to_write, since vfs uses this to
				 * get # of written pages.
				 */
				if (ret == AOP_WRITEPAGE_ACTIVATE) {
					ret = 0;
					goto next;
				} else if (ret == -EAGAIN) {
					ret = 0;
					if (wbc->sync_mode == WB_SYNC_ALL) {
						f2fs_io_schedule_timeout(
							DEFAULT_IO_TIMEOUT);
						goto retry_write;
					}
					goto next;
				}
				done_index = folio_next_index(folio);
				done = 1;
				break;
			}

			if (wbc->nr_to_write <= 0 &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
next:
			if (need_readd)
				goto readd;
		}
		release_pages(pages, nr_pages);
		cond_resched();
	}
#ifdef CONFIG_F2FS_FS_COMPRESSION
	/* flush remained pages in compress cluster */
	if (f2fs_compressed_file(inode) && !f2fs_cluster_is_empty(&cc)) {
		ret = f2fs_write_multi_pages(&cc, &submitted, wbc, io_type);
		nwritten += submitted;
		wbc->nr_to_write -= submitted;
		if (ret) {
			done = 1;
			retry = 0;
		}
	}
	if (f2fs_compressed_file(inode))
		f2fs_destroy_compress_ctx(&cc, false);
#endif
	if (retry) {
		index = 0;
		end = -1;
		goto retry;
	}
	if (wbc->range_cyclic && !done)
		done_index = 0;
	if (wbc->range_cyclic || (range_whole && wbc->nr_to_write > 0))
		mapping->writeback_index = done_index;

	if (nwritten)
		f2fs_submit_merged_write_cond(F2FS_M_SB(mapping), mapping->host,
					      NULL, 0, DATA);
	/* submit cached bio of IPU write */
	if (bio)
		f2fs_submit_merged_ipu_write(sbi, &bio, NULL);

#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (pages != pages_local)
		kfree(pages);
#endif

	return ret;
}
```
我们可以把所有有关压缩的代码暂时去除来简化一下代码的逻辑,简化之后代码如下:
```C
static int f2fs_write_cache_pages(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	int ret = 0;
	int done = 0, retry = 0;
	/*int done = 0, retry = 0;: 
	声明整型变量 done 和 retry 并初始化为 0。
	done: 可能用作循环控制标志，当设置为非 0 值时，可能表示回写操作完成或需要提前结束。
	retry: 可能用作重试标志，当设置为非 0 值时，可能表示需要重试某些操作。*/
	struct page *pages_local[F2FS_ONSTACK_PAGES];
	/*声明一个 page 结构体指针数组 pages_local，大小为 F2FS_ONSTACK_PAGES。
	这很可能是一个 栈上分配的 局部页数组，用于临时存储要写入的页。 
	F2FS_ONSTACK_PAGES 应该是一个宏定义，表示栈上页数组的最大容量。*/
	struct page **pages = pages_local;
	/*声明一个 page 结构体二级指针 pages，并初始化为指向 pages_local 数组。 
	pages 指针将用于后续操作，这样可以根据需要动态地切换到堆上分配的页数组*/
	struct folio_batch fbatch;
	struct f2fs_sb_info *sbi = F2FS_M_SB(mapping);
	struct bio *bio = NULL;
	sector_t last_block;
	int nr_folios, p, idx;
	int nr_pages;
	unsigned int max_pages = F2FS_ONSTACK_PAGES;
	pgoff_t index;
	pgoff_t end; /* Inclusive */
	pgoff_t done_index;
	int range_whole = 0;
	xa_mark_t tag;
	int nwritten = 0;
	int submitted = 0;
	int i;
	folio_batch_init(&fbatch);

	if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);

	if (wbc->range_cyclic) {
		/*如果为真，表示回写范围是 循环的 (cyclic)。 
		循环回写通常用于后台周期性的回写操作，从上次停止的位置继续回写。*/
		index = mapping->writeback_index; /* prev offset
		很可能存储了 上次循环回写操作结束时的页偏移量，作为本次循环回写的起始位置。 */
		end = -1;
	} else {
		index = wbc->range_start >> PAGE_SHIFT;
		/*将 wbc->range_start (回写起始位置) 右移 PAGE_SHIFT 位，
		得到起始页的索引，并赋值给 index。*/
		end = wbc->range_end >> PAGE_SHIFT;
		/*将 wbc->range_end (回写结束位置) 右移 PAGE_SHIFT 位，
		得到结束页的索引，并赋值给 end。*/
		if (wbc->range_start == 0 && wbc->range_end == LLONG_MAX)
			range_whole = 1;
		/*条件判断，检查回写范围是否是 整个文件或地址空间。
		如果回写范围是整个文件，则将 range_whole 标志设置为 1。*/
	}
	if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
		tag = PAGECACHE_TAG_TOWRITE;
	else
		tag = PAGECACHE_TAG_DIRTY;
	while (!done && !retry && (index <= end)) 
	{
		/*
		....
		....
		收集nr_pages的循环
		*/
write: /*开始write循环*/
		for (i = 0; i < nr_pages; i++)
		{
			struct page *page = pages[i];
			struct folio *folio = page_folio(page);
			/*看来是把page再转化回folio
			实现上存在疑问 因为这样似乎是之前把所有的folio拆散现在却又单个page转化回去
			*/
			bool need_readd;
readd:
			need_readd = false;
			/* give a priority to WB_SYNC threads */
			if (atomic_read(&sbi->wb_sync_req[DATA]) &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
			done_index =
				folio->index; /*更新已完成的index为folio的起始index*/
retry_write:
			/*为了保证并发下的安全性,我们必须在锁的保护下把之前所有检查过的事项全再检查一遍*/
			folio_lock(folio);
			/*
								在锁的保护下检查folio是否还在内存中了
								*/
			if (unlikely(folio->mapping != mapping)) {
continue_unlock:
				folio_unlock(folio); /*不在的话就解锁continue*/
				continue;
			}

			if (!folio_test_dirty(folio)) {
				/* someone wrote it for us */
				goto continue_unlock;
			}

			if (folio_test_writeback(folio)) {
				if (wbc->sync_mode == WB_SYNC_NONE)
					goto continue_unlock;
				f2fs_wait_on_page_writeback(&folio->page, DATA,
							    true, true);
			}

			if (!folio_clear_dirty_for_io(folio))
				goto continue_unlock;
			ret = f2fs_write_single_data_page(folio, &submitted,
							  &bio, &last_block,
							  wbc, io_type, 0,
							  true);
			if (ret == AOP_WRITEPAGE_ACTIVATE)
				folio_unlock(folio);
#ifdef CONFIG_F2FS_FS_COMPRESSION
result:
#endif
			nwritten += submitted;
			wbc->nr_to_write -= submitted;

			if (unlikely(ret)) {
				/*
				 * keep nr_to_write, since vfs uses this to
				 * get # of written pages.
				 */
				if (ret == AOP_WRITEPAGE_ACTIVATE) {
					ret = 0;
					goto next;
				} else if (ret == -EAGAIN) {
					ret = 0;
					if (wbc->sync_mode == WB_SYNC_ALL) {
						f2fs_io_schedule_timeout(
							DEFAULT_IO_TIMEOUT);
						goto retry_write;
					}
					goto next;
				}
				done_index = folio_next_index(folio);
				done = 1;
				break;
			}

			if (wbc->nr_to_write <= 0 &&
			    wbc->sync_mode == WB_SYNC_NONE) {
				done = 1;
				break;
			}
next:
			if (need_readd)
				goto readd;
		}
	}
}
```
static int f2fs_write_cache_folios(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	struct iomap_writepage_ctx wpc={
		.io_type = io_type,
	}
#ifdef CONFIG_F2FS_FS_COMPRESSION
	struct inode *inode = mapping->host;
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = NULL_CLUSTER,
		.rfolios = NULL,
		.nr_rpages = 0,
        .last_idx=0,
        .nr_remains=F2FS_I(inode)->i_cluster_size,
		.cfolios = NULL,
		.valid_nr_cpages = 0,
		.rbuf = NULL,
		.cbuf = NULL,
		.rlen = PAGE_SIZE * F2FS_I(inode)->i_cluster_size,
		.private = NULL,
	};
#endif
    struct folio* folio=NULL;
    int error;
    if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);
    /*我们来说一下writeback_iter都干了什么事情啊。
    首先index根据是否是回环写这个判断已经写好了并且我们也不用计算结束索引,也不用手动进行folio_batch_init。然后if (wbc->sync_mode 
    == WB_SYNC_ALL || wbc->tagged_writepages)
		tag_pages_for_writeback(mapping, index, end);这个逻辑是一模一样的,少了else
		tag = PAGECACHE_TAG_DIRTY这个分支。
        然后 关于等待folio写回以及其是为脏的这些同步问题,writeback_iter内部已经帮我们解决了*/
    while(folio=writeback_iter(mapping,wbc,folio,&error))
    {
        /*这里开始就和f2fs_write_cache_pages原先的逻辑有很大的区别了
        在考虑和truncate并发的时候,原f2fs_write_cache_pages函数是将从
        filemap_get_folios_tag拿到的folio拆散成page,然后再对它们一一上锁。
        并且检查此时mapping还在不在了。但是writeback_iter这里,我们一次只能看到一整个folio,所有的并发了,等待回写操作了,全是只基于这个folio单元的。因此我们上来就直接对这单个folio做并发上的检查*/
    struct f2fs_iomap_folio_state *ifs = folio->private;
	struct inode *inode = folio->mapping->host;
	u64 pos = folio_pos(folio);
	u64 end_pos = pos + folio_size(folio);
	u64 end_aligned = 0;
	unsigned count = 0;
	int error = 0;
	u32 rlen;

	WARN_ON_ONCE(!folio_test_locked(folio));
	WARN_ON_ONCE(folio_test_dirty(folio));
	WARN_ON_ONCE(folio_test_writeback(folio));
	if (!iomap_writepage_handle_eof(folio, inode, &end_pos)) {
		folio_unlock(folio);
		return 0;/*iomap_writepage_map这里return 0实际上相当于循环continue一次*/
	}
    folio_start_writeback(folio);
    end_aligned = round_up(end_pos, i_blocksize(inode));
    }
    /*我们预期的f2fs_iomap_find_dirty_range函数无论是寻找脏的cluster 还是寻找脏的page区间,每次只会返回一段连续的脏区间。*/
	while ((rlen = f2fs_iomap_find_dirty_range(folio, &pos, end_aligned))) {
		error = f2fs_iomap_writepage_map_blocks(wpc,wbc, folio, inode,
				pos, end_pos, rlen, &count);
		/*我感觉可以把io_type给存到writepage_ctx之中*/
		/*begin f2fs_iomap_writepage_map_blocks*/
		/*注意我们已经处理了eof*/
		/*第一步 计算实际能映射得到的区间并且我打算在这一步就将fwpc中的iomap_end中的bio分配好。*/
		error = wpc->ops->map_blocks(wpc, inode, pos, dirty_len);
		/*begin wpc->ops->map_blocks*/
		struct f2fs_map_blocks map;
		block_t start_blk=F2FS_BYTES_TO_BLK(pos);
		block_t len_blks=F2FS_BYTES_TO_BLK(offset + length - 1) - start_blk + 1;
		err=f2fs_map_blocks_iomap(start_blk,len_blks,&map);
		
		if (error)
			break;
		pos += rlen;
	}
	
}

static int iomap_writepage_map(struct iomap_writepage_ctx *wpc,
		struct writeback_control *wbc, struct folio *folio)
{
	struct iomap_folio_state *ifs = folio->private;
	struct inode *inode = folio->mapping->host;
	u64 pos = folio_pos(folio);
	u64 end_pos = pos + folio_size(folio);
	u64 end_aligned = 0;
	unsigned count = 0;
	int error = 0;
	u32 rlen;

	WARN_ON_ONCE(!folio_test_locked(folio));
	WARN_ON_ONCE(folio_test_dirty(folio));
	WARN_ON_ONCE(folio_test_writeback(folio));

	trace_iomap_writepage(inode, pos, folio_size(folio));

	if (!iomap_writepage_handle_eof(folio, inode, &end_pos)) {
		folio_unlock(folio);
		return 0;
	}
	WARN_ON_ONCE(end_pos <= pos);

	if (i_blocks_per_folio(inode, folio) > 1) {
		if (!ifs) {
			ifs = ifs_alloc(inode, folio, 0);
			iomap_set_range_dirty(folio, 0, end_pos - pos);
		}

		/*
		 * Keep the I/O completion handler from clearing the writeback
		 * bit until we have submitted all blocks by adding a bias to
		 * ifs->write_bytes_pending, which is dropped after submitting
		 * all blocks.
		 */
		WARN_ON_ONCE(atomic_read(&ifs->write_bytes_pending) != 0);
		atomic_inc(&ifs->write_bytes_pending);
	}

	/*
	 * Set the writeback bit ASAP, as the I/O completion for the single
	 * block per folio case happen hit as soon as we're submitting the bio.
	 */
	folio_start_writeback(folio);

	/*
	 * Walk through the folio to find dirty areas to write back.
	 */
	end_aligned = round_up(end_pos, i_blocksize(inode));
	while ((rlen = iomap_find_dirty_range(folio, &pos, end_aligned))) {
		error = iomap_writepage_map_blocks(wpc, wbc, folio, inode,
				pos, end_pos, rlen, &count);
		if (error)
			break;
		pos += rlen;
	}

	if (count)
		wpc->nr_folios++;

	/*
	 * We can have dirty bits set past end of file in page_mkwrite path
	 * while mapping the last partial folio. Hence it's better to clear
	 * all the dirty bits in the folio here.
	 */
	iomap_clear_range_dirty(folio, 0, folio_size(folio));

	/*
	 * Usually the writeback bit is cleared by the I/O completion handler.
	 * But we may end up either not actually writing any blocks, or (when
	 * there are multiple blocks in a folio) all I/O might have finished
	 * already at this point.  In that case we need to clear the writeback
	 * bit ourselves right after unlocking the page.
	 */
	folio_unlock(folio);
	if (ifs) {
		if (atomic_dec_and_test(&ifs->write_bytes_pending))
			folio_end_writeback(folio);
	} else {
		if (!count)
			folio_end_writeback(folio);
	}
	mapping_set_error(inode->i_mapping, error);
	return error;
}
 static int iomap_add_to_ioend(struct iomap_writepage_ctx *wpc,
		struct writeback_control *wbc, struct folio *folio,
		struct inode *inode, loff_t pos, loff_t end_pos,
		unsigned len)
{
	struct iomap_folio_state *ifs = folio->private;
	size_t poff = offset_in_folio(folio, pos);
	unsigned int ioend_flags = 0;
	int error;

	if (wpc->iomap.type == IOMAP_UNWRITTEN)
		ioend_flags |= IOMAP_IOEND_UNWRITTEN;
	if (wpc->iomap.flags & IOMAP_F_SHARED)
		ioend_flags |= IOMAP_IOEND_SHARED;
	if (pos == wpc->iomap.offset && (wpc->iomap.flags & IOMAP_F_BOUNDARY))
		ioend_flags |= IOMAP_IOEND_BOUNDARY;

	if (!wpc->ioend || !iomap_can_add_to_ioend(wpc, pos, ioend_flags)) {
new_ioend:
		error = iomap_submit_ioend(wpc, 0);
		if (error)
			return error;
		wpc->ioend = iomap_alloc_ioend(wpc, wbc, inode, pos,
				ioend_flags);
	}

	if (!bio_add_folio(&wpc->ioend->io_bio, folio, len, poff))
		goto new_ioend;

	if (ifs)
		atomic_add(len, &ifs->write_bytes_pending);

	/*
	 * Clamp io_offset and io_size to the incore EOF so that ondisk
	 * file size updates in the ioend completion are byte-accurate.
	 * This avoids recovering files with zeroed tail regions when
	 * writeback races with appending writes:
	 *
	 *    Thread 1:                  Thread 2:
	 *    ------------               -----------
	 *    write [A, A+B]
	 *    update inode size to A+B
	 *    submit I/O [A, A+BS]
	 *                               write [A+B, A+B+C]
	 *                               update inode size to A+B+C
	 *    <I/O completes, updates disk size to min(A+B+C, A+BS)>
	 *    <power failure>
	 *
	 *  After reboot:
	 *    1) with A+B+C < A+BS, the file has zero padding in range
	 *       [A+B, A+B+C]
	 *
	 *    |<     Block Size (BS)   >|
	 *    |DDDDDDDDDDDD0000000000000|
	 *    ^           ^        ^
	 *    A          A+B     A+B+C
	 *                       (EOF)
	 *
	 *    2) with A+B+C > A+BS, the file has zero padding in range
	 *       [A+B, A+BS]
	 *
	 *    |<     Block Size (BS)   >|<     Block Size (BS)    >|
	 *    |DDDDDDDDDDDD0000000000000|00000000000000000000000000|
	 *    ^           ^             ^           ^
	 *    A          A+B           A+BS       A+B+C
	 *                             (EOF)
	 *
	 *    D = Valid Data
	 *    0 = Zero Padding
	 *
	 * Note that this defeats the ability to chain the ioends of
	 * appending writes.
	 */
	wpc->ioend->io_size += len;
	if (wpc->ioend->io_offset + wpc->ioend->io_size > end_pos)
		wpc->ioend->io_size = end_pos - wpc->ioend->io_offset;

	wbc_account_cgroup_owner(wbc, folio, len);
	return 0;
}
我现在试图用iomap的思路正在重写f2fs的脏页回写但是我在思路上遇到了困境 我使用iomap的目的是想更高效地查找数据块 在wbc提供的map_blocks函数指针中我们是可以利用f2fs_map	blocks 直接查找一个连续的块区间的 然后iomap find dirtu range 能告诉我们一个folio 中脏的区间 具体来说 开头是 调整过的pos 结尾是 end
f2fs目前现有的代码只有这种单个page写回的函数 我想增加一个f2fs_write_single_folio
这个函数的接口呢 除了有folio以外 我觉得要传一个从folio写的起点位置 和终点位置 然后 是否是压缩块 io type 我觉得可以
