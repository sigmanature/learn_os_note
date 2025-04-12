这是 F2FS 实现逻辑的地方，该逻辑在数据可以复制到页缓存 folio 以进行缓冲写入之前需要执行。
**相关函数**

*   [prepare_atomic_write_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/prepare_atomic_write_begin.md)

*   [prepare_write_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/prepare_write_begin.md)
```c
// f2fs_write_begin: F2FS 对 address_space_operations->write_begin 的实现
static int f2fs_write_begin(struct file *file, struct address_space *mapping,
		loff_t pos, unsigned len, struct folio **foliop, void **fsdata)
{
	struct inode *inode = mapping->host; // 从地址空间获取 inode
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode); // 获取 F2FS 超级块信息
	struct folio *folio; // 指向目标 folio 的指针
	pgoff_t index = pos >> PAGE_SHIFT; // 从文件偏移量计算页索引
	bool need_balance = false; // 标志：是否需要调用 f2fs_balance_fs？（空间清理）
	bool use_cow = false; // 标志：是否正在使用写时复制（用于原子写入）？
	// block_t 是 F2FS 的块地址类型。
	// 初始化为 NULL_ADDR (0)，表示块地址尚不知道/未确定。
	block_t blkaddr = NULL_ADDR;
	int err = 0; // 错误码

	// 追踪操作的开始
	trace_f2fs_write_begin(inode, pos, len);

	// 检查文件系统是否准备好进行新的写入（检查点稳定）
	if (!f2fs_is_checkpoint_ready(sbi)) {
		err = -ENOSPC; // 如果检查点正在进行或文件系统已满/错误，则无法写入
		goto fail;
	}

	/*
	 * 我们应该在此刻检查这个，以避免 inode 页和 #0 页上的死锁。
	 * 内联数据转换的锁定规则应该是：
	 * folio_lock(folio #0) -> folio_lock(inode_page)
	 */
	// 如果写入超出第一页（index > 0），确保内联数据（如果有的话）
	// 已转换为普通块，以避免稍后复杂的锁定。
	if (index != 0) {
		err = f2fs_convert_inline_inode(inode);
		if (err)
			goto fail;
	}

#ifdef CONFIG_F2FS_FS_COMPRESSION
	// 特别处理压缩文件
	if (f2fs_compressed_file(inode)) {
		int ret;
		struct page *page; // 对压缩 API 使用 page*

		*fsdata = NULL; // fsdata 由压缩逻辑使用

		// 如果写入的是整页并且不是原子写入，可能会跳过压缩准备？
		// 或者也许压缩对整页的处理方式不同。跳转到 repeat。
		if (len == PAGE_SIZE && !(f2fs_is_atomic_file(inode)))
			goto repeat; // 先正常获取 folio

		// 准备覆盖压缩数据。这可能会分配资源
		// 或决定写入是否可以直接进行。
		// 'fsdata' 可能会在这里为 write_end 填充。
		ret = f2fs_prepare_compress_overwrite(inode, &page,
							index, fsdata);
		if (ret < 0) { // 压缩准备期间出错
			err = ret;
			goto fail;
		} else if (ret) { // 压缩准备处理了页面设置 (ret == 1)
			// 如果 ret > 0，表示页面已准备好（例如，已解压）。
			*foliop = page_folio(page); // 返回准备好的 folio
			return 0; // 成功，跳过正常的 folio 处理
		}
		// 如果 ret == 0，则继续下面的正常 folio 处理。
	}
#endif

repeat: // 用于重试获取 folio 的标签
	/*
	 * 不要使用 FGP_STABLE 来避免死锁。
	 * 将在下面使用我们的 IO 控制来等待。
	 * FGP_LOCK: 锁定 folio。
	 * FGP_WRITE: 写入意图。
	 * FGP_CREAT: 如果 folio 不存在则创建。
	 * GFP_NOFS: 内存分配标志，不能递归进入文件系统。
	 */
	// 从页缓存获取 folio。如果不存在则创建，并锁定它。
	folio = __filemap_get_folio(mapping, index,
				FGP_LOCK | FGP_WRITE | FGP_CREAT, GFP_NOFS);
	if (IS_ERR(folio)) { // 获取/创建 folio 失败（例如，OOM）
		err = PTR_ERR(folio);
		goto fail;
	}

	/* TODO: 由于与 .writepage 的竞争，簇可能被压缩 */
	// 一条注释，指出与压缩可能存在的竞争条件。

	*foliop = folio; // 返回获取到的 folio 指针

	// 判断是原子写入还是普通写入，并调用相应的辅助函数
	// 来查找/预留块地址并检查元数据是否需要平衡。
	if (f2fs_is_atomic_file(inode))
		// 原子写入使用写时复制 (CoW)
		err = prepare_atomic_write_begin(sbi, folio, pos, len,
					&blkaddr, &need_balance, &use_cow);
	else
		// 普通写入
		err = prepare_write_begin(sbi, folio, pos, len,
					&blkaddr, &need_balance);

	if (err) // 来自 prepare_*_write_begin 的错误
		goto put_folio; // 释放 folio

	// 如果 prepare 函数指示需要元数据平衡（例如，节点块分配）
	// 并且启用了配额并且空闲空间不足，则触发后台清理。
	if (need_balance && !IS_NOQUOTA(inode) &&
			has_not_enough_free_secs(sbi, 0, 0)) {
		folio_unlock(folio); // 临时解锁 folio 以允许清理
		f2fs_balance_fs(sbi, true); // 运行同步清理/平衡
		folio_lock(folio); // 重新获取 folio 锁
		// 检查在解锁期间 folio 是否被截断/移除
		if (folio->mapping != mapping) {
			/* folio 在我们解锁时被截断了 */
			folio_unlock(folio);
			folio_put(folio); // 释放过时的 folio
			goto repeat; // 重试获取 folio
		}
	}

	// 等待此页面上任何正在进行的回写 IO 完成。
	// DATA: 等待数据回写。false: 不强制等待。true: 可能排队 IO。
	f2fs_wait_on_page_writeback(&folio->page, DATA, false, true);

	/* --- 处理 folio 内容（如果需要，读取现有数据）--- */
	// 如果当前写入覆盖整个 folio 或者 folio 已经是 up-to-date
	// （意味着其内容准确反映磁盘或更新），那么我们不需要
	// 从磁盘读取任何内容。从用户空间的复制将覆盖所有需要的部分。
	if (len == folio_size(folio) || folio_test_uptodate(folio))
		return 0; // 准备好接收用户数据复制

	// --- 对非 up-to-date 的 folio 进行部分写入 ---
	// 我们需要 folio 中*未*被这次写入覆盖的部分的现有数据。

	// 优化：如果从页面开始写入（pos 是页对齐的）
	// 并且这次写入超出了当前文件末尾（i_size）
	// 并且 verity 未激活（verity 需要读取以进行验证）
	// 那么我们可以只将 folio 中当前写入*之后*的部分清零。
	// 正在写入的部分无论如何都会被用户数据覆盖。
	if (!(pos & (PAGE_SIZE - 1)) && (pos + len) >= i_size_read(inode) &&
	    !f2fs_verity_in_progress(inode)) {
		folio_zero_segment(folio, len, folio_size(folio)); // 从写入结束到 folio 结束清零
		return 0; // 准备好接收用户数据复制
	}

	// --- 对非 up-to-date folio 进行部分写入的一般情况 ---
	// 我们需要从磁盘获取现有的块内容。
	// 'blkaddr' 由 prepare_*_write_begin 确定。

	// NEW_ADDR 是一个特殊值，表示为此页索引保留/分配了一个新块
	// （它以前是一个空洞或是正在进行 CoW）。
	if (blkaddr == NEW_ADDR) {
		// 如果是新块，则没有现有数据可读。
		// 在复制部分用户数据之前将整个 folio 清零。
		folio_zero_segment(folio, 0, folio_size(folio));
		folio_mark_uptodate(folio); // 将其标记为 up-to-date（逻辑上已清零）
	} else {
		// 该块已存在于磁盘上（blkaddr 是有效的物理地址）。
		// 读取前检查块地址是否有效。
		if (!f2fs_is_valid_blkaddr(sbi, blkaddr,
				DATA_GENERIC_ENHANCE_READ)) {
			err = -EFSCORRUPTED; // 检测到文件系统损坏
			goto put_folio;
		}
		// 提交一个异步读请求以获取现有块内容。
		// 如果 use_cow 为 true（原子写入），则从 CoW inode 的块映射中读取。
		err = f2fs_submit_page_read(use_cow ?
				F2FS_I(inode)->cow_inode : inode,
				folio, blkaddr, 0, true);
		if (err) // 提交读请求出错
			goto put_folio;

		// 重新锁定 folio（submit_page_read 可能短暂解锁？）- VFS 页面读取模式通常涉及此操作。
		folio_lock(folio);
		// 再次检查在读取提交期间可能解锁时 folio 是否被截断。
		if (unlikely(folio->mapping != mapping)) {
			folio_unlock(folio);
			folio_put(folio);
			goto repeat; // 重试获取 folio
		}
		// 检查读取是否成功完成（folio 现在是 up-to-date）。
		if (unlikely(!folio_test_uptodate(folio))) {
			err = -EIO; // 读取失败
			goto put_folio;
		}
	}
	// Folio 现在已锁定且 up-to-date（要么已清零，要么从磁盘读取）。
	return 0; // 成功，准备好进行 copy_from_iter

put_folio: // 错误路径：解锁并释放 folio 引用
	folio_unlock(folio);
	folio_put(folio);
fail: // 错误路径：标记写入失败以进行潜在的回滚/清理
	f2fs_write_failed(inode, pos + len);
	return err; // 返回错误码
}
```