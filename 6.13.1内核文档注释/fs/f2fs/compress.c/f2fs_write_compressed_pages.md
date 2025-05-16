```C
static int f2fs_write_compressed_pages(struct compress_ctx *cc,
					int *submitted,
					struct writeback_control *wbc,
					enum iostat_type io_type)
{
	struct inode *inode = cc->inode; // 获取关联的 inode
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode); // 获取超级块信息
	struct f2fs_inode_info *fi = F2FS_I(inode); // 获取 F2FS 特定的 inode 信息
	struct f2fs_io_info fio = { // 初始化 f2fs_io_info 结构体，用于描述 I/O 操作
		.sbi = sbi, // 超级块信息
		.ino = cc->inode->i_ino, // inode 号
		.type = DATA, // I/O 类型：数据
		.op = REQ_OP_WRITE,/*这个多半是提供给bio层的标志吧*/ // I/O 操作：写。是的，这是传递给 bio 层的操作类型标志。
		.op_flags = wbc_to_write_flags(wbc), // 根据 writeback_control 转换的 I/O 标志
		.old_blkaddr = NEW_ADDR, // 旧的块地址，这里初始化为 NEW_ADDR，表示将分配新块
		.page = NULL, // 原始数据页，在循环中会设置
		.encrypted_page = NULL, // 加密后的页，如果加密启用
		.compressed_page = NULL, // 压缩后的页，如果加密未启用
		.io_type = io_type, // I/O 统计类型
		.io_wbc = wbc, // writeback_control 结构体
		.encrypted = fscrypt_inode_uses_fs_layer_crypto(cc->inode) ?
									1 : 0, // 检查 inode 是否使用文件系统层加密
	};
	struct folio *folio; // folio 结构体，用于获取页信息
	struct dnode_of_data dn; // dnode 结构体，用于定位数据块地址
	struct node_info ni; // node_info 结构体，用于获取节点信息（如版本）
	struct compress_io_ctx *cic; // 压缩 I/O 上下文结构体，用于跟踪整个压缩簇的写回状态
	pgoff_t start_idx = start_idx_of_cluster(cc); // 计算当前压缩簇的起始页索引
	/*先算一个簇的起始idx*/
	unsigned int last_index = cc->cluster_size - 1; // 计算簇中最后一页的索引
	loff_t psize; // 文件大小
	int i, err; // 循环变量和错误码
	bool quota_inode = IS_NOQUOTA(inode); // 检查是否是配额 inode

	/* we should bypass data pages to proceed the kworker jobs */
	// 如果检查点错误发生，设置 inode 的 mapping 错误标志，并跳转到清理
	if (unlikely(f2fs_cp_error(sbi))) {
		mapping_set_error(inode->i_mapping, -EIO);
		goto out_free;
	}

	// 处理锁：如果是配额 inode，需要获取 node_write 读锁，避免在检查点期间分配块
	if (quota_inode) {
		/*
		 * We need to wait for node_write to avoid block allocation during
		 * checkpoint. This can only happen to quota writes which can cause
		 * the below discard race condition.
		 */
		f2fs_down_read(&sbi->node_write);
	} else if (!f2fs_trylock_op(sbi)) { // 否则尝试获取一般的操作锁
		goto out_free; // 获取失败则跳转到清理
	}

	// 初始化 dnode 结构体，用于定位起始页的数据块地址
	set_new_dnode(&dn, cc->inode, NULL, NULL, 0);

	// 获取起始页对应的 dnode
	err = f2fs_get_dnode_of_data(&dn, start_idx, LOOKUP_NODE);
	if (err)
		goto out_unlock_op; // 获取失败则跳转到解锁并清理

	// 检查簇中的所有页是否都有对应的块地址（即没有空洞）。
	// 对于压缩簇，理论上不应该有空洞，这个检查可能更多是防御性的。
	for (i = 0; i < cc->cluster_size; i++) {
		if (data_blkaddr(dn.inode, dn.node_page,
					dn.ofs_in_node + i) == NULL_ADDR)/*有空洞就要直接out put dnode 这有点值得在意吧*/
			goto out_put_dnode; // 如果发现空洞，释放 dnode 并跳转清理
	}

	// 获取簇中最后一页的 folio，用于计算文件大小
	folio = page_folio(cc->rpages[last_index]);
	psize = folio_pos(folio) + folio_size(folio); // 计算文件当前大小

	// 获取 dnode 对应的节点信息，主要是为了获取版本号
	err = f2fs_get_node_info(fio.sbi, dn.nid, &ni, false);
	if (err)
		goto out_put_dnode; // 获取失败则释放 dnode 并跳转清理

	fio.version = ni.version; // 设置 fio 的版本号

	/*开始分配cic*/
	// 分配 compress_io_ctx (cic) 结构体
	cic = f2fs_kmem_cache_alloc(cic_entry_slab, GFP_F2FS_ZERO, false, sbi);
	if (!cic)
		goto out_put_dnode; // 分配失败则释放 dnode 并跳转清理

	// 初始化 cic
	cic->magic = F2FS_COMPRESSED_PAGE_MAGIC; // 设置魔数
	cic->inode = inode; // 关联 inode
	// 设置待处理的页数，即有效压缩页的数量。这个计数会在每个压缩块的 I/O 完成时递减。
	atomic_set(&cic->pending_pages, cc->valid_nr_cpages);
	// 为 cic 分配原始页数组，用于在 I/O 完成时处理原始页（如解锁）
	cic->rpages = page_array_alloc(cc->inode, cc->cluster_size);
	if (!cic->rpages)
		goto out_put_cic; // 分配失败则释放 cic 并跳转清理

	cic->nr_rpages = cc->cluster_size;/*直接nr_rpages*/ // 设置原始页数组大小，等于簇大小

	// 准备压缩/加密页：将它们与 cic 关联，并处理加密
	for (i = 0; i < cc->valid_nr_cpages; i++) {
		// 将压缩页与 inode、对应的原始页索引和 cic 关联起来
		f2fs_set_compressed_page(cc->cpages[i], inode,
				page_folio(cc->rpages[i + 1])->index, cic);
		/*还是要注意,cc里面的page都是*/ // 注意：cc->cpages 存储的是压缩/加密后的数据页
		fio.compressed_page = cc->cpages[i]; // 将当前压缩页设置到 fio 中

		// 获取当前压缩页对应的原始页在文件中的旧块地址
		fio.old_blkaddr = data_blkaddr(dn.inode, dn.node_page,
						dn.ofs_in_node + i + 1); // +1 是因为簇的第一个块是头，数据从索引 1 开始

		/* wait for GCed page writeback via META_MAPPING */
		// 等待旧块地址的写回完成，这可能发生在 GC 过程中
		f2fs_wait_on_block_writeback(inode, fio.old_blkaddr);

		// 如果启用加密
		if (fio.encrypted) {
			fio.page = cc->rpages[i + 1]; // 设置原始页
			err = f2fs_encrypt_one_page(&fio); // 对原始页进行加密，结果存入 fio.encrypted_page
			if (err)
				goto out_destroy_crypt; // 加密失败则跳转清理
			cc->cpages[i] = fio.encrypted_page; // 更新 cc->cpages 为加密后的页
		}
	}

	// 标记整个簇正在进行写回
	set_cluster_writeback(cc);

	// 将 cc 的原始页复制到 cic 的原始页数组中
	for (i = 0; i < cc->cluster_size; i++)
		cic->rpages[i] = cc->rpages[i];

	// 循环处理簇中的每个块槽位 (0 到 cluster_size - 1)
	for (i = 0; i < cc->cluster_size; i++, dn.ofs_in_node++) {
		block_t blkaddr; // 当前槽位的块地址

		// 获取当前槽位对应的原始页
		fio.page = cc->rpages[i];
		// 获取当前槽位在文件中的旧块地址
		blkaddr = f2fs_data_blkaddr(&dn);
		fio.old_blkaddr = blkaddr; // 设置 fio 的旧块地址

		/* cluster header */
		// 处理簇头 (槽位 0)
		if (i == 0) {
			// 如果旧块地址是 COMPRESS_ADDR，说明之前就是压缩簇，统计压缩块数量
			if (blkaddr == COMPRESS_ADDR)
				fio.compr_blocks++;
			// 如果旧块地址是有效的数据块地址，则将其标记为无效（因为要写新块）
			if (__is_valid_data_blkaddr(blkaddr))
				f2fs_invalidate_blocks(sbi, blkaddr, 1);
			// 更新 dnode 中的块地址为 COMPRESS_ADDR，表示这是一个压缩簇的头
			f2fs_update_data_blkaddr(&dn, COMPRESS_ADDR);
			goto unlock_continue; // 跳转到解锁原始页并继续循环
		}

		// 对于非头槽位，如果 fio.compr_blocks 大于 0 (说明旧簇是压缩的)，且当前旧块地址有效，则统计压缩块数量
		if (fio.compr_blocks && __is_valid_data_blkaddr(blkaddr))
			fio.compr_blocks++;

		// 处理有效压缩页之后的槽位 (空闲槽位)
		if (i > cc->valid_nr_cpages) {
			// 如果旧块地址有效，则将其标记为无效，并更新 dnode 中的块地址为 NEW_ADDR
			if (__is_valid_data_blkaddr(blkaddr)) {
				f2fs_invalidate_blocks(sbi, blkaddr, 1);
				f2fs_update_data_blkaddr(&dn, NEW_ADDR);
			}
			goto unlock_continue; // 跳转到解锁原始页并继续循环
		}

		// 对于有效压缩页对应的槽位 (1 到 valid_nr_cpages)
		// 确保旧块地址不是 NULL_ADDR (不应该是空洞)
		f2fs_bug_on(fio.sbi, blkaddr == NULL_ADDR);

		// 设置 fio 中的压缩/加密页。注意索引是 i - 1，因为 cc->cpages 只有 valid_nr_cpages 个，对应槽位 1 到 valid_nr_cpages
		if (fio.encrypted)
			fio.encrypted_page = cc->cpages[i - 1];
		else
			fio.compressed_page = cc->cpages[i - 1];

		// 将 cc->cpages[i - 1] 设置为 NULL，表示该页的所有权已转移到 fio/I/O 子系统
		cc->cpages[i - 1] = NULL;
		fio.submitted = 0; // 初始化 submitted 标志
		// 调用核心函数进行数据写回。这会分配新块，构建 bio，提交 I/O。
		f2fs_outplace_write_data(&dn, &fio);
		// 检查 I/O 是否成功提交
		if (unlikely(!fio.submitted)) {
			// 如果提交失败，取消簇写回标记，清理 cic，并跳转到清理加密页
			cancel_cluster_writeback(cc, cic, i);

			/* To call fscrypt_finalize_bounce_page */
			i = cc->valid_nr_cpages; // 设置 i 为 valid_nr_cpages，以便清理循环能处理所有已准备的加密页
			*submitted = 0; // 设置 submitted 计数为 0
			goto out_destroy_crypt; // 跳转到清理加密页
		}
		(*submitted)++; // 成功提交，增加提交计数
unlock_continue:
		// 递减 inode 的脏页计数
		inode_dec_dirty_pages(cc->inode);
		// 解锁原始页。注意这里解锁的是 fio.page，即 cc->rpages[i]。
		unlock_page(fio.page);
	}

	// 更新 inode 的压缩块统计信息
	if (fio.compr_blocks)
		f2fs_i_compr_blocks_update(inode, fio.compr_blocks - 1, false); // 减去旧的压缩块数量 (头不算)
	f2fs_i_compr_blocks_update(inode, cc->valid_nr_cpages, true); // 加上新的压缩块数量
	add_compr_block_stat(inode, cc->valid_nr_cpages); // 更新全局压缩块统计

	// 设置 inode 的 FI_APPEND_WRITE 标志，表示有数据被写入
	set_inode_flag(cc->inode, FI_APPEND_WRITE);

	// 释放 dnode
	f2fs_put_dnode(&dn);
	// 释放锁
	if (quota_inode)
		f2fs_up_read(&sbi->node_write);
	else
		f2fs_unlock_op(sbi);

	// 更新 inode 的 last_disk_size
	spin_lock(&fi->i_size_lock);
	if (fi->last_disk_size < psize)
		fi->last_disk_size = psize;
	spin_unlock(&fi->i_size_lock);

	// 释放 cc 中的原始页数组
	f2fs_put_rpages(cc);
	// 释放 cc 中的压缩/加密页数组
	page_array_free(cc->inode, cc->cpages, cc->nr_cpages);
	cc->cpages = NULL; // 将指针置空
	// 销毁 compress_ctx
	f2fs_destroy_compress_ctx(cc, false);
	return 0; // 成功返回

// 错误清理路径：清理加密页
out_destroy_crypt:
	// 释放 cic 中的原始页数组
	page_array_free(cc->inode, cic->rpages, cc->cluster_size);

	// 循环清理已准备好的加密页
	for (--i; i >= 0; i--) {
		if (!cc->cpages[i]) // 如果页已经被转移 (成功提交的部分)，跳过
			continue;
		// 调用 fscrypt 的清理函数
		fscrypt_finalize_bounce_page(&cc->cpages[i]);
	}
// 错误清理路径：释放 cic
out_put_cic:
	kmem_cache_free(cic_entry_slab, cic);
// 错误清理路径：释放 dnode
out_put_dnode:
	f2fs_put_dnode(&dn);
// 错误清理路径：释放锁
out_unlock_op:
	if (quota_inode)
		f2fs_up_read(&sbi->node_write);
	else
		f2fs_unlock_op(sbi);
// 错误清理路径：释放 cc 中的压缩/加密页
out_free:
	for (i = 0; i < cc->valid_nr_cpages; i++) {
		// 释放压缩页的内存
		f2fs_compress_free_page(cc->cpages[i]);
		cc->cpages[i] = NULL; // 将指针置空
	}
	// 释放 cc 中的压缩/加密页数组
	page_array_free(cc->inode, cc->cpages, cc->nr_cpages);
	cc->cpages = NULL; // 将指针置空
	return -EAGAIN; // 返回错误码
}
```
