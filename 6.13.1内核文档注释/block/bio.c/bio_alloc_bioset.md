```C
/**
 * bio_alloc_bioset - allocate a bio for I/O
 * @bdev:	block device to allocate the bio for (can be %NULL)
 * @nr_vecs:	number of bvecs to pre-allocate
 * @opf:	operation and flags for bio
 * @gfp_mask:   the GFP_* mask given to the slab allocator
 * @bs:		the bio_set to allocate from.
 *
 * Allocate a bio from the mempools in @bs.
 *
 * If %__GFP_DIRECT_RECLAIM is set then bio_alloc will always be able to
 * allocate a bio.  This is due to the mempool guarantees.  To make this work,
 * callers must never allocate more than 1 bio at a time from the general pool.
 * Callers that need to allocate more than 1 bio must always submit the
 * previously allocated bio for IO before attempting to allocate a new one.
 * Failure to do so can cause deadlocks under memory pressure.
 *
 * Note that when running under submit_bio_noacct() (i.e. any block driver),
 * bios are not submitted until after you return - see the code in
 * submit_bio_noacct() that converts recursion into iteration, to prevent
 * stack overflows.
 *
 * This would normally mean allocating multiple bios under submit_bio_noacct()
 * would be susceptible to deadlocks, but we have
 * deadlock avoidance code that resubmits any blocked bios from a rescuer
 * thread.
 *
 * However, we do not guarantee forward progress for allocations from other
 * mempools. Doing multiple allocations from the same mempool under
 * submit_bio_noacct() should be avoided - instead, use bio_set's front_pad
 * for per bio allocations.
 *
 * Returns: Pointer to new bio on success, NULL on failure.
 */
struct bio *bio_alloc_bioset(struct block_device *bdev, unsigned short nr_vecs,
			     blk_opf_t opf, gfp_t gfp_mask,
			     struct bio_set *bs)
{
	gfp_t saved_gfp = gfp_mask;
	struct bio *bio;
	void *p;

	/* should not use nobvec bioset for nr_vecs > 0 */
	if (WARN_ON_ONCE(!mempool_initialized(&bs->bvec_pool) && nr_vecs > 0))
		return NULL;

	if (opf & REQ_ALLOC_CACHE) {
		if (bs->cache && nr_vecs <= BIO_INLINE_VECS) {
			bio = bio_alloc_percpu_cache(bdev, nr_vecs, opf,
						     gfp_mask, bs);
			if (bio)
				return bio;
			/*
			 * No cached bio available, bio returned below marked with
			 * REQ_ALLOC_CACHE to particpate in per-cpu alloc cache.
			 */
		} else {
			opf &= ~REQ_ALLOC_CACHE;
		}
	}

	/*
	 * submit_bio_noacct() converts recursion to iteration; this means if
	 * we're running beneath it, any bios we allocate and submit will not be
	 * submitted (and thus freed) until after we return.
	 *
	 * This exposes us to a potential deadlock if we allocate multiple bios
	 * from the same bio_set() while running underneath submit_bio_noacct().
	 * If we were to allocate multiple bios (say a stacking block driver
	 * that was splitting bios), we would deadlock if we exhausted the
	 * mempool's reserve.
	 *
	 * We solve this, and guarantee forward progress, with a rescuer
	 * workqueue per bio_set. If we go to allocate and there are bios on
	 * current->bio_list, we first try the allocation without
	 * __GFP_DIRECT_RECLAIM; if that fails, we punt those bios we would be
	 * blocking to the rescuer workqueue before we retry with the original
	 * gfp_flags.
	 */
	if (current->bio_list &&
	    (!bio_list_empty(&current->bio_list[0]) ||
	     !bio_list_empty(&current->bio_list[1])) &&
	    bs->rescue_workqueue)
		gfp_mask &= ~__GFP_DIRECT_RECLAIM;

	p = mempool_alloc(&bs->bio_pool, gfp_mask);
	if (!p && gfp_mask != saved_gfp) {
		punt_bios_to_rescuer(bs);
		gfp_mask = saved_gfp;
		p = mempool_alloc(&bs->bio_pool, gfp_mask);
	}
	if (unlikely(!p))
		return NULL;
	if (!mempool_is_saturated(&bs->bio_pool))
		opf &= ~REQ_ALLOC_CACHE;

	bio = p + bs->front_pad;
	if (nr_vecs > BIO_INLINE_VECS) {
		struct bio_vec *bvl = NULL;

		bvl = bvec_alloc(&bs->bvec_pool, &nr_vecs, gfp_mask);
		if (!bvl && gfp_mask != saved_gfp) {
			punt_bios_to_rescuer(bs);
			gfp_mask = saved_gfp;
			bvl = bvec_alloc(&bs->bvec_pool, &nr_vecs, gfp_mask);
		}
		if (unlikely(!bvl))
			goto err_free;

		bio_init(bio, bdev, bvl, nr_vecs, opf);
	} else if (nr_vecs) {
		bio_init(bio, bdev, bio->bi_inline_vecs, BIO_INLINE_VECS, opf);
	} else {
		bio_init(bio, bdev, NULL, 0, opf);
	}

	bio->bi_pool = bs;
	return bio;

err_free:
	mempool_free(p, &bs->bio_pool);
	return NULL;
}
```