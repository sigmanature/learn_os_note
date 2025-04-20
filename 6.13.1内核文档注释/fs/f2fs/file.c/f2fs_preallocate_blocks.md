```C
/*
* Preallocate blocks for a write request, if it is possible and helpful to do
* so.  Returns a positive number if blocks may have been preallocated, 0 if no
* blocks were preallocated, or a negative errno value if something went
* seriously wrong.  Also sets FI_PREALLOCATED_ALL on the inode if *all* the
* requested blocks (not just some of them) have been allocated.
*/
static int f2fs_preallocate_blocks(struct kiocb *iocb, struct iov_iter *iter,
   			   bool dio)
{
   struct inode *inode = file_inode(iocb->ki_filp); /* Get the inode from the file object associated with the kiocb */
   struct f2fs_sb_info *sbi = F2FS_I_SB(inode); /* Get the F2FS superblock info from the inode */
   const loff_t pos = iocb->ki_pos; /* Get the starting position of the write operation from the kiocb */
   const size_t count = iov_iter_count(iter); /* Get the total number of bytes to be written from the iov_iter */
   struct f2fs_map_blocks map = {}; /* Initialize a f2fs_map_blocks structure to describe the block mapping request */
   int flag; /* Flag to indicate the type of block mapping request (pre-AIO or pre-DIO) */
   int ret; /* Return value to store the result of block mapping operations */

   /* If it will be an out-of-place direct write, don't bother. */
   if (dio && f2fs_lfs_mode(sbi))
   	return 0; /* In LFS mode (Log-structured File System), direct I/O writes are often out-of-place,
   	        * so preallocation might not be beneficial and could even be counterproductive.
   	        * Return 0 to indicate no preallocation was done. */
   /*
    * Don't preallocate holes aligned to DIO_SKIP_HOLES which turns into
    * buffered IO, if DIO meets any holes.
    */
   if (dio && i_size_read(inode) &&
   	(F2FS_BYTES_TO_BLK(pos) < F2FS_BLK_ALIGN(i_size_read(inode))))
   	return 0; /* For Direct I/O, if the write starts before the aligned end of the current i_size
   	        * and there's existing data (i_size_read(inode) > 0), it might indicate a hole is being partially
   	        * overwritten. In such cases, preallocation for DIO might lead to buffered IO behavior due to DIO_SKIP_HOLES
   	        * handling holes. To avoid unexpected buffered IO, skip preallocation in this scenario. */

   /* No-wait I/O can't allocate blocks. */
   if (iocb->ki_flags & IOCB_NOWAIT)
   	return 0; /* If the I/O operation is non-blocking (IOCB_NOWAIT flag is set), block allocation (which might block)
   	        * is not allowed. Return 0 to indicate no preallocation. */

   /* If it will be a short write, don't bother. */
   if (fault_in_iov_iter_readable(iter, count))
   	return 0; /* `fault_in_iov_iter_readable` checks if all data in the iov_iter is already in memory (e.g., from page cache).
   	        * If it is, it means the write operation might be very short or even a no-op (data already there).
   	        * In such cases, preallocation is likely not necessary. Return 0. */

   if (f2fs_has_inline_data(inode)) {
   	/* If the data will fit inline, don't bother. */
   	if (pos + count <= MAX_INLINE_DATA(inode))
   		return 0; /* If the total size of the write (current position + write count) is still within the inline data limit
   		        * for this inode, then inline data is sufficient, and no block preallocation is needed. Return 0. */
   	ret = f2fs_convert_inline_inode(inode); /* If the write will exceed the inline data limit, we need to convert the inode
   	                                        * from inline data to regular block-based data storage. Call f2fs_convert_inline_inode
   	                                        * to perform this conversion. */
   	if (ret)
   		return ret; /* If the inline inode conversion fails, return the error code. */
   }

   /* Do not preallocate blocks that will be written partially in 4KB. */
   map.m_lblk = F2FS_BLK_ALIGN(pos); /* Calculate the starting logical block number aligned to 4KB boundary based on the write position.
                                       * F2FS_BLK_ALIGN(pos) effectively rounds up 'pos' to the nearest 4KB boundary in blocks. 很奇怪为什么这个lblk要进行上取整*/
   map.m_len = F2FS_BYTES_TO_BLK(pos + count); /* Calculate the ending logical block number (in blocks) for the write range. */
   if (map.m_len > map.m_lblk)
   	map.m_len -= map.m_lblk; /* Calculate the length of the preallocation range in blocks.
   	                         * It's the difference between the end block and the aligned start block.
   	                         * This effectively calculates the number of blocks needed for preallocation, starting from the
   	                         * 4KB aligned position up to the end of the write range. */
   else
   	return 0; /* If the calculated length is not positive (meaning write is within or before the aligned start block),
   	        * no preallocation is needed. Return 0. */

   if (!IS_DEVICE_ALIASING(inode))
   	map.m_may_create = true; /* If device aliasing is not active for this inode (likely a regular file on a non-aliasing device),
   	                         * set m_may_create to true, allowing block allocation to create new blocks if needed. */
   if (dio) {
   	map.m_seg_type = f2fs_rw_hint_to_seg_type(sbi,
   					inode->i_write_hint); /* For Direct I/O, set the segment type for block allocation based on the inode's write hint.
   					                                 * This helps the allocator choose appropriate segments for DIO writes. */
   	flag = F2FS_GET_BLOCK_PRE_DIO; /* Set the flag to indicate this is a preallocation request for Direct I/O. */
   } else {
   	map.m_seg_type = NO_CHECK_TYPE; /* For buffered I/O, set segment type to NO_CHECK_TYPE, meaning no specific segment type preference. */
   	flag = F2FS_GET_BLOCK_PRE_AIO; /* Set the flag to indicate this is a preallocation request for buffered I/O (AIO - Asynchronous I/O in general context here). */
   }

   ret = f2fs_map_blocks(inode, &map, flag); /* Call f2fs_map_blocks to perform the actual block preallocation.
                                             * Pass the inode, the map structure describing the request, and the flag indicating the type of request. */
   /* -ENOSPC|-EDQUOT are fine to report the number of allocated blocks. */
   if (ret < 0 && !((ret == -ENOSPC || ret == -EDQUOT) && map.m_len > 0))
   	return ret; /* If f2fs_map_blocks returns an error, and it's not -ENOSPC (No Space) or -EDQUOT (Quota Exceeded)
   	        * while some blocks were actually allocated (map.m_len > 0), then return the error.
   	        * -ENOSPC and -EDQUOT are treated specially because even if preallocation fails due to space or quota limits,
   	        * some blocks might still have been successfully preallocated, and we want to return the count of those. */
   if (ret == 0)
   	set_inode_flag(inode, FI_PREALLOCATED_ALL); /* If f2fs_map_blocks returns 0 (success), it means all requested blocks were preallocated.
   	                                            * Set the FI_PREALLOCATED_ALL inode flag to indicate this. */
   return map.m_len; /* Return the number of blocks that were actually preallocated (which might be less than requested if space/quota limits were hit,
                      * or equal to the requested length if successful, or 0 if no preallocation was attempted or possible). */
}

```