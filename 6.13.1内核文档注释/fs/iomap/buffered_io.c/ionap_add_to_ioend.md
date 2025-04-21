```C
/*
 * iomap_add_to_ioend - Add a mapped folio segment to an I/O end structure.
 * @wpc: Context structure for iomap writepage operations.
 * @wbc: Writeback control structure.
 * @folio: The folio containing the data to add.
 * @inode: The inode the data belongs to.
 * @pos: Starting byte position of the segment within the file.
 * @end_pos: The effective end position of the folio (adjusted for EOF).
 * @len: Length of the segment to add.
 *
 * This function attempts to add a segment of a folio (representing mapped,
 * dirty data) to the current `iomap_ioend`'s bio structure (`io_bio`) stored
 * in the writepage context (`wpc`).
 *
 * If no `ioend` exists, or if the current segment cannot be merged with the
 * existing `ioend` (checked by `iomap_can_add_to_ioend`), it first submits
 * the existing `ioend` (if any) and then allocates a new one.
 *
 * It handles updating the pending write count for multi-block folios (`ifs`)
 * and clamps the `ioend` size to the actual end-of-file to ensure correct
 * on-disk file size updates upon I/O completion.
 *
 * Return: 0 on success, or a negative error code on failure (e.g., if
 *         submitting the previous ioend failed).
 */
static int iomap_add_to_ioend(struct iomap_writepage_ctx *wpc,
		struct writeback_control *wbc, struct folio *folio,
		struct inode *inode, loff_t pos, loff_t end_pos,
		unsigned len)
{
	// Get the per-folio state if it exists (for multi-block folios).
	struct iomap_folio_state *ifs = folio->private;
	// Calculate the offset of the segment within the folio's data buffer.
	size_t poff = offset_in_folio(folio, pos);
	int error; // Variable to store potential errors.

	// Check if we need a new ioend structure. This happens if:
	// 1. There is no current ioend cached in the context (!wpc->ioend).
	// 2. The current segment cannot be merged with the existing ioend
	//    (checked by iomap_can_add_to_ioend based on contiguity, flags, etc.).
	if (!wpc->ioend || !iomap_can_add_to_ioend(wpc, pos)) {
// Label for the 'create new ioend' logic path.
new_ioend:
		// Submit the *currently cached* ioend (if one exists).
		// Pass 0 as error since we are just finishing a batch, not handling a failure here.
		error = iomap_submit_ioend(wpc, 0);
		// If submitting the previous ioend failed, propagate the error.
		if (error)
			return error;
		// Allocate a new ioend structure, initializing it for the current
		// inode and position. Store it in the writepage context.
		wpc->ioend = iomap_alloc_ioend(wpc, wbc, inode, pos);
		// Note: iomap_alloc_ioend likely returns NULL on allocation failure,
		// which would cause a crash later. Error handling might be needed here.
	}

	// Attempt to add the folio segment (specified by folio, len, poff)
	// to the bio structure within the current ioend.
	// bio_add_folio returns false if the segment cannot be added (e.g., bio full).
	if (!bio_add_folio(&wpc->ioend->io_bio, folio, len, poff))
		// If adding failed (bio full), we need to submit the current ioend
		// and start a new one. Jump back to the new_ioend label.
		goto new_ioend;

	// If per-folio state exists (multi-block folio), atomically add the length
	// of this segment to the count of bytes pending writeback for this folio.
	// This counter helps track completion of all blocks within the folio.
	if (ifs)
		atomic_add(len, &ifs->write_bytes_pending);

	/*
	 * Update the total size tracked by the ioend.
	 * Then, clamp the ioend's effective size (`io_size`) so that the
	 * total range (`io_offset + io_size`) does not exceed the actual,
	 * potentially EOF-adjusted, end position (`end_pos`) of the data
	 * being written from this folio.
	 * This prevents the I/O completion handler from potentially updating the
	 * on-disk file size based on a block boundary that extends beyond the
	 * actual end of the file data, which could lead to zero-padding issues
	 * if writeback races with appending writes or truncates.
	 */
	// Increment the logical size of data within this ioend batch.
	wpc->ioend->io_size += len;
	// Check if the calculated end of the ioend batch exceeds the actual end of data.
	if (wpc->ioend->io_offset + wpc->ioend->io_size > end_pos)
		// If it exceeds, clamp io_size to match the actual end position.
		wpc->ioend->io_size = end_pos - wpc->ioend->io_offset;

	// Account this write operation against the cgroup owner of the folio.
	wbc_account_cgroup_owner(wbc, folio, len);
	// Return 0 indicating success in adding the segment to an ioend.
	return 0;
}

/**
 * iomap_can_add_to_ioend - Check if a mapping can be merged into the current ioend.
 * @wpc: Context structure holding the current ioend and the new mapping details.
 * @pos: Starting byte position of the new mapping being considered.
 *
 * Determines if the block mapping currently stored in `wpc->iomap` (which
 * starts at file offset `pos`) can be added to the existing `ioend`'s bio
 * (`wpc->ioend->io_bio`). Merging is possible only if several conditions related
 * to contiguity, flags, type, and batch size limits are met.
 *
 * Return: true if the mapping can be added, false otherwise.
 */
static bool iomap_can_add_to_ioend(struct iomap_writepage_ctx *wpc, loff_t pos)
{
	// Check 1: Boundary condition. If the new mapping starts exactly where
	// the previous one ended (`wpc->iomap.offset == pos`) BUT the new
	// mapping is marked as a boundary (`IOMAP_F_BOUNDARY`), we cannot merge
	// across this boundary.
	if (wpc->iomap.offset == pos && (wpc->iomap.flags & IOMAP_F_BOUNDARY))
		return false; // Cannot merge across explicit boundary

	// Check 2: Shared flag consistency. If the shared status (`IOMAP_F_SHARED`)
	// differs between the new mapping and the existing ioend, they cannot be
	// part of the same bio/ioend.
	if ((wpc->iomap.flags & IOMAP_F_SHARED) !=
	    (wpc->ioend->io_flags & IOMAP_F_SHARED))
		return false; // Shared status mismatch

	// Check 3: Mapping type consistency. If the type of the new mapping
	// (e.g., MAPPED, UNWRITTEN) differs from the type of the existing ioend,
	// they cannot be merged.
	if (wpc->iomap.type != wpc->ioend->io_type)
		return false; // Mapping type mismatch

	// Check 4: Logical contiguity. The starting position (`pos`) of the new
	// mapping must exactly follow the end position of the data currently in
	// the ioend (`wpc->ioend->io_offset + wpc->ioend->io_size`).
	if (pos != wpc->ioend->io_offset + wpc->ioend->io_size)
		return false; // Not logically contiguous

	// Check 5: Physical contiguity. The starting sector of the new mapping
	// on disk (`iomap_sector(...)`) must exactly follow the ending sector
	// of the bio currently built in the ioend (`bio_end_sector(...)`).
	if (iomap_sector(&wpc->iomap, pos) !=
	    bio_end_sector(&wpc->ioend->io_bio))
		return false; // Not physically contiguous on disk

	/*
	 * Check 6: Batch size limit. To prevent excessively long bio chains
	 * (which can increase I/O completion latency and potentially hold folio
	 * locks for too long), limit the number of folios that can be added
	 * to a single ioend/bio.
	 */
	if (wpc->nr_folios >= IOEND_BATCH_SIZE)
		return false; // Reached batch size limit

	// If all checks passed, the mapping can be added to the current ioend.
	return true;
}

/*
 * iomap_submit_ioend - Submit the bio associated with the current ioend.
 * @wpc: Context structure holding the ioend to submit.
 * @error: Error code from preceding operations (0 if none).
 *
 * Submits the bio (`io_bio`) contained within the `iomap_ioend` currently
 * cached in `wpc->ioend`. Before submission, it calls the filesystem's
 * `prepare_ioend` callback (if provided) to allow final adjustments or
 * completion handler setup.
 *
 * If an error is present (either passed in or returned by `prepare_ioend`),
 * it manually triggers the bio's endio handler with the error status instead
 * of submitting it for actual I/O. This ensures proper cleanup (like clearing
 * writeback flags) even on failure paths.
 *
 * After submission or error handling, it clears the `wpc->ioend` pointer.
 *
 * Return: The final error status (0 on successful submission initiation,
 *         or the relevant negative error code).
 */
static int iomap_submit_ioend(struct iomap_writepage_ctx *wpc, int error)
{
	// If there is no cached ioend, there's nothing to submit.
	// Return the error status passed in (or 0 if none).
	if (!wpc->ioend)
		return error;

	/*
	 * Allow the filesystem to perform final preparations before I/O submission.
	 * This hook might attach filesystem-specific completion data or modify flags.
	 * This is called even if 'error' is already set, to allow FS cleanup.
	 */
	if (wpc->ops->prepare_ioend)
		// Call the FS callback, potentially updating the error status.
		error = wpc->ops->prepare_ioend(wpc->ioend, error);

	// Check if there's an error *after* the prepare step.
	if (error) {
		// An error occurred either before this function or in prepare_ioend.
		// Do not submit the bio for I/O. Instead, simulate I/O completion
		// with an error status.
		// Convert the Linux errno to a block layer status code.
		wpc->ioend->io_bio.bi_status = errno_to_blk_status(error);
		// Manually call the bio's end I/O handler. This will trigger the
		// completion callback chain (including iomap's and the FS's),
		// ensuring flags like PG_writeback are cleared and errors are processed.
		bio_endio(&wpc->ioend->io_bio);
	} else {
		// No errors, proceed with submitting the bio to the block layer
		// for asynchronous I/O. Completion will be handled by the
		// bio's bi_end_io callback when the I/O finishes.
		submit_bio(&wpc->ioend->io_bio);
	}

	// Clear the cached ioend pointer in the context, as it has now been handled
	// (either submitted or error-completed).
	wpc->ioend = NULL;
	// Return the final error status.
	return error;
}

/* Definition of iomap_writepage_ctx struct (provided for context) */
struct iomap_writepage_ctx {
	struct iomap		iomap; // Holds details of the *current* block mapping
	struct iomap_ioend	*ioend; // Pointer to the *current* I/O batch being built
	const struct iomap_writeback_ops *ops; // Filesystem-specific callbacks
	u32			nr_folios;	/* folios added to the current ioend */
};

/* Definition of iomap_ioend struct (provided for context) */
struct iomap_ioend {
	struct list_head	io_list;	/* (Not used in this snippet) next ioend in chain */
	u16			io_type; // Type of mapping (e.g., MAPPED, UNWRITTEN)
	u16			io_flags;	/* IOMAP_F_* flags (e.g., SHARED) */
	struct inode		*io_inode;	/* file being written to */
	size_t			io_size;	/* size of valid data within file bounds in this ioend */
	loff_t			io_offset;	/* starting file offset of this ioend */
	sector_t		io_sector;	/* starting disk sector (likely unused here, bio has it) */
	struct bio		io_bio;		/* The actual bio structure for I/O - MUST BE LAST! */
};

/* Definition of submit_bio function (provided for context) */
/**
 * submit_bio - submit a bio to the block device layer for I/O
 * @bio: The &struct bio which describes the I/O
 *
 * submit_bio() is used to submit I/O requests to block devices. It is passed a
 * fully set up &struct bio that describes the I/O that needs to be done. The
 * bio will be send to the device described by the bi_bdev field.
 *
 * The success/failure status of the request, along with notification of
 * completion, is delivered asynchronously through the ->bi_end_io() callback
 * in @bio. The bio must NOT be touched by the caller until ->bi_end_io() has
 * been called.
 */
// void submit_bio(struct bio *bio); // Actual function signature
```