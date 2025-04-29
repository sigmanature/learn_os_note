```C
static struct bio *f2fs_grab_read_bio(struct inode *inode, block_t blkaddr,
				      unsigned nr_pages, blk_opf_t op_flag,
				      pgoff_t first_idx, bool for_write)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct bio *bio;
	struct bio_post_read_ctx *ctx = NULL;/*这个后处理的上下文将是一个难点*/
	unsigned int post_read_steps = 0;
	sector_t sector;
	struct block_device *bdev = f2fs_target_device(sbi, blkaddr, &sector);/*
    * 首先处理多设备的情况,如果是单设备的话,根据传进来的数据块地址转为扇区地址
    */

	bio = bio_alloc_bioset(bdev, bio_max_segs(nr_pages),
			       REQ_OP_READ | op_flag,
			       for_write ? GFP_NOIO : GFP_KERNEL, &f2fs_bioset);
    /*分配实际的bio。重点就在这里。被f2fs_read_multi_pages调用的时候啊,直接分配的就是最大数量
    的bio了*/
	if (!bio)
		return ERR_PTR(-ENOMEM);
	bio->bi_iter.bi_sector = sector;/*记录要读的起始扇区位置 其实iomap也是费尽心机想得到这个*/
	f2fs_set_bio_crypt_ctx(bio, inode, first_idx, NULL, GFP_NOFS);
	bio->bi_end_io = f2fs_read_end_io;

	if (fscrypt_inode_uses_fs_layer_crypto(inode))
		post_read_steps |= STEP_DECRYPT;

	if (f2fs_need_verity(inode, first_idx))
		post_read_steps |= STEP_VERITY;

	/*
	 * STEP_DECOMPRESS is handled specially, since a compressed file might
	 * contain both compressed and uncompressed clusters.  We'll allocate a
	 * bio_post_read_ctx if the file is compressed, but the caller is
	 * responsible for enabling STEP_DECOMPRESS if it's actually needed.
	 */

	if (post_read_steps || f2fs_compressed_file(inode)) {
		/* Due to the mempool, this never fails. */
		ctx = mempool_alloc(bio_post_read_ctx_pool, GFP_NOFS);/*这个上下文还是在堆上分配的*/
		ctx->bio = bio;
		ctx->sbi = sbi;
		ctx->enabled_steps = post_read_steps;
		ctx->fs_blkaddr = blkaddr;
		ctx->decompression_attempted = false;
		bio->bi_private = ctx;  
	}
	iostat_alloc_and_bind_ctx(sbi, bio, ctx);

	return bio;
}
```