```C
static int f2fs_write_compressed_pages(struct compress_ctx *cc,
					int *submitted,
					struct writeback_control *wbc,
					enum iostat_type io_type)
{
	struct inode *inode = cc->inode;
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct f2fs_inode_info *fi = F2FS_I(inode);
	struct f2fs_io_info fio = {
		.sbi = sbi,
		.ino = cc->inode->i_ino,
		.type = DATA,
		.op = REQ_OP_WRITE,/*这个多半是提供给bio层的标志吧*/
		.op_flags = wbc_to_write_flags(wbc),
		.old_blkaddr = NEW_ADDR,
		.page = NULL,
		.encrypted_page = NULL,
		.compressed_page = NULL,
		.io_type = io_type,
		.io_wbc = wbc,
		.encrypted = fscrypt_inode_uses_fs_layer_crypto(cc->inode) ?
									1 : 0,
	};
	struct folio *folio;
	struct dnode_of_data dn;
	struct node_info ni;
	struct compress_io_ctx *cic;
	pgoff_t start_idx = start_idx_of_cluster(cc);
	unsigned int last_index = cc->cluster_size - 1;
	loff_t psize;
	int i, err;
	bool quota_inode = IS_NOQUOTA(inode);

	/* we should bypass data pages to proceed the kworker jobs */
	if (unlikely(f2fs_cp_error(sbi))) {
		mapping_set_error(inode->i_mapping, -EIO);
		goto out_free;
	}

	if (quota_inode) {
		/*
		 * We need to wait for node_write to avoid block allocation during
		 * checkpoint. This can only happen to quota writes which can cause
		 * the below discard race condition.
		 */
		f2fs_down_read(&sbi->node_write);
	} else if (!f2fs_trylock_op(sbi)) {
		goto out_free;
	}

	set_new_dnode(&dn, cc->inode, NULL, NULL, 0);

	err = f2fs_get_dnode_of_data(&dn, start_idx, LOOKUP_NODE);
	if (err)
		goto out_unlock_op;

	for (i = 0; i < cc->cluster_size; i++) {
		if (data_blkaddr(dn.inode, dn.node_page,
					dn.ofs_in_node + i) == NULL_ADDR)/*有空洞就要直接out put dnode 这有点值得在意吧*/
			goto out_put_dnode;
	}

	folio = page_folio(cc->rpages[last_index]);
	psize = folio_pos(folio) + folio_size(folio);

	err = f2fs_get_node_info(fio.sbi, dn.nid, &ni, false);
	if (err)
		goto out_put_dnode;

	fio.version = ni.version;

	cic = f2fs_kmem_cache_alloc(cic_entry_slab, GFP_F2FS_ZERO, false, sbi);
	if (!cic)
		goto out_put_dnode;

	cic->magic = F2FS_COMPRESSED_PAGE_MAGIC;
	cic->inode = inode;
	atomic_set(&cic->pending_pages, cc->valid_nr_cpages);
	cic->rpages = page_array_alloc(cc->inode, cc->cluster_size);
	if (!cic->rpages)
		goto out_put_cic;

	cic->nr_rpages = cc->cluster_size;/*直接nr_rpages*/

	for (i = 0; i < cc->valid_nr_cpages; i++) {
		f2fs_set_compressed_page(cc->cpages[i], inode,
				page_folio(cc->rpages[i + 1])->index, cic);
		fio.compressed_page = cc->cpages[i];

		fio.old_blkaddr = data_blkaddr(dn.inode, dn.node_page,
						dn.ofs_in_node + i + 1);

		/* wait for GCed page writeback via META_MAPPING */
		f2fs_wait_on_block_writeback(inode, fio.old_blkaddr);

		if (fio.encrypted) {
			fio.page = cc->rpages[i + 1];
			err = f2fs_encrypt_one_page(&fio);
			if (err)
				goto out_destroy_crypt;
			cc->cpages[i] = fio.encrypted_page;
		}
	}

	set_cluster_writeback(cc);

	for (i = 0; i < cc->cluster_size; i++)
		cic->rpages[i] = cc->rpages[i];

	for (i = 0; i < cc->cluster_size; i++, dn.ofs_in_node++) {
		block_t blkaddr;

		blkaddr = f2fs_data_blkaddr(&dn);
		fio.page = cc->rpages[i];
		fio.old_blkaddr = blkaddr;

		/* cluster header */
		if (i == 0) {
			if (blkaddr == COMPRESS_ADDR)
				fio.compr_blocks++;
			if (__is_valid_data_blkaddr(blkaddr))
				f2fs_invalidate_blocks(sbi, blkaddr, 1);
			f2fs_update_data_blkaddr(&dn, COMPRESS_ADDR);
			goto unlock_continue;
		}

		if (fio.compr_blocks && __is_valid_data_blkaddr(blkaddr))
			fio.compr_blocks++;

		if (i > cc->valid_nr_cpages) {
			if (__is_valid_data_blkaddr(blkaddr)) {
				f2fs_invalidate_blocks(sbi, blkaddr, 1);
				f2fs_update_data_blkaddr(&dn, NEW_ADDR);
			}
			goto unlock_continue;
		}

		f2fs_bug_on(fio.sbi, blkaddr == NULL_ADDR);

		if (fio.encrypted)
			fio.encrypted_page = cc->cpages[i - 1];
		else
			fio.compressed_page = cc->cpages[i - 1];

		cc->cpages[i - 1] = NULL;
		fio.submitted = 0;
		f2fs_outplace_write_data(&dn, &fio);
		if (unlikely(!fio.submitted)) {
			cancel_cluster_writeback(cc, cic, i);

			/* To call fscrypt_finalize_bounce_page */
			i = cc->valid_nr_cpages;
			*submitted = 0;
			goto out_destroy_crypt;
		}
		(*submitted)++;
unlock_continue:
		inode_dec_dirty_pages(cc->inode);
		unlock_page(fio.page);
	}

	if (fio.compr_blocks)
		f2fs_i_compr_blocks_update(inode, fio.compr_blocks - 1, false);
	f2fs_i_compr_blocks_update(inode, cc->valid_nr_cpages, true);
	add_compr_block_stat(inode, cc->valid_nr_cpages);

	set_inode_flag(cc->inode, FI_APPEND_WRITE);

	f2fs_put_dnode(&dn);
	if (quota_inode)
		f2fs_up_read(&sbi->node_write);
	else
		f2fs_unlock_op(sbi);

	spin_lock(&fi->i_size_lock);
	if (fi->last_disk_size < psize)
		fi->last_disk_size = psize;
	spin_unlock(&fi->i_size_lock);

	f2fs_put_rpages(cc);
	page_array_free(cc->inode, cc->cpages, cc->nr_cpages);
	cc->cpages = NULL;
	f2fs_destroy_compress_ctx(cc, false);
	return 0;

out_destroy_crypt:
	page_array_free(cc->inode, cic->rpages, cc->cluster_size);

	for (--i; i >= 0; i--) {
		if (!cc->cpages[i])
			continue;
		fscrypt_finalize_bounce_page(&cc->cpages[i]);
	}
out_put_cic:
	kmem_cache_free(cic_entry_slab, cic);
out_put_dnode:
	f2fs_put_dnode(&dn);
out_unlock_op:
	if (quota_inode)
		f2fs_up_read(&sbi->node_write);
	else
		f2fs_unlock_op(sbi);
out_free:
	for (i = 0; i < cc->valid_nr_cpages; i++) {
		f2fs_compress_free_page(cc->cpages[i]);
		cc->cpages[i] = NULL;
	}
	page_array_free(cc->inode, cc->cpages, cc->nr_cpages);
	cc->cpages = NULL;
	return -EAGAIN;
}
```
