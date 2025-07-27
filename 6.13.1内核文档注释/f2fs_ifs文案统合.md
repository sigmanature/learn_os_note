## 功能背景
保持不变
## 重大问题分析
我们已经在问题背景中分析过f2fs_iomap_folio_state的必要性 没有它导致的问题触发现象,就是我们在任何需要使用f2fs_iomap_folio_state地方的代码,以及iomap框架中需要访问iomap_folio_state地方的代码,会将其误认为是一个指针值 导致内核在形式上抛出空指针解引用的异常出来。而实际我们将我们的f2fs_iomap_folio_state集成到我们的代码的时候,我发现真是牵一发而动全身。任何原先使用旧的set_page_private/get_page_private的代码路径全部都需要更改。这里面涉及到的代码路径不光有普通数据folio 还有元数据folio中存储自己private字段的地方也需要替换成我们新设计的接口。
举个例子，page cache中的f2fs_release_folio,f2fs_is_compressed_folio. 我们顺带一下它们的函数调用栈 用dump stack来画
同时，我最终还惊讶的发现，初赛时我认为本可以和iomap框架自身保持一致地不为0阶folio分配f2fs_iomap_folio_state，然而 在我遭遇了下面这个问题之后,我的想法彻底改变了:
（问题显现)
```C
move_data_page
static int move_data_page(struct inode *inode, block_t bidx, int gc_type,
						unsigned int segno, int off)
{
	struct folio *folio;
	int err = 0;

	folio = f2fs_get_lock_data_folio(inode, bidx, true);
	if (IS_ERR(folio))
		return PTR_ERR(folio);

	if (!check_valid_map(F2FS_I_SB(inode), segno, off)) {
		err = -ENOENT;
		goto out;
	}

	err = f2fs_gc_pinned_control(inode, gc_type, segno);
	if (err)
		goto out;

	if (gc_type == BG_GC) {
		if (folio_test_writeback(folio)) {
			err = -EAGAIN;
			goto out;
		}
		folio_mark_dirty(folio);
		set_page_private_gcing(&folio->page);
	} else {
		struct f2fs_io_info fio = {
			.sbi = F2FS_I_SB(inode),
			.ino = inode->i_ino,
			.type = DATA,
			.temp = COLD,
			.op = REQ_OP_WRITE,
			.op_flags = REQ_SYNC,
			.old_blkaddr = NULL_ADDR,
			.page = &folio->page,
			.encrypted_page = NULL,
			.need_lock = LOCK_REQ,
			.io_type = FS_GC_DATA_IO,
		};
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
out:
	f2fs_folio_put(folio, true);
	return err;
}
```
（下面我们详细地描述一下整个gc进程先触发 设置了folio的private字段 然后我们发现在iomap_buffered_write之中我们永远不能像我们在自己使用f2fs_iomap_folio_state的时候 区分出来这里面是一个单纯的private flag字段而不是一个指针 因此是必定会引起空指针解引用崩溃的）
## 方案设计 (这里面再细分若干个小节但是目录那里可以不展示出来吧)
首先f2fs_ifs结构体的各种定义是保持不变的。但是我我们描述f2fsfolio private api的时候
要进行一个较大的改动了。大体的讲法:取我们的任何一个f2fs_get/set_folio_private作为示例讲法。
讲解到我们不直接依赖于folio的阶数判断 而是先直接test标志位 因为这里面巧就巧在 f2fs原先的标志位设计本来就是能区分是指针还是不是指针
毕竟已经有了no_pointer了 而f2fs_iomap_folio_state又恰好是个指针
然后我们展现我们的f2fs_iomap_folio_state在各个代码路径里的具体使用,把他们给展示出来。
另外我们大致展开描述一下f2fs对page private flag的几个动作项。就是设置,获取，清除 我们稍微展开描述一下。
