
```C
/**
* f2fs_convert_inline_page - Convert inline data page to normal data page
* @dn: dnode of data, representing the data node of the inode
* @page: page to write data to (the folio will be extracted from this page)
*
* This function converts an inode that currently uses inline data to store its content
* to use regular data blocks instead. This is typically done when the inline data
* capacity is exceeded, and the file needs to grow beyond the inline limit.
*/
int f2fs_convert_inline_page(struct dnode_of_data *dn, struct page *page)
{
   struct f2fs_io_info fio = {
   	.sbi = F2FS_I_SB(dn->inode), /* Get superblock info from inode */
   	.ino = dn->inode->i_ino,     /* Get inode number */
   	.type = DATA,                /* Operation type is DATA write */
   	.op = REQ_OP_WRITE,          /* Request operation is WRITE */
   	.op_flags = REQ_SYNC | REQ_PRIO, /* Request flags: synchronous and high priority */
   	.page = page,                /* Page to write to */
   	.encrypted_page = NULL,      /* No encrypted page involved in this operation */
   	.io_type = FS_DATA_IO,        /* File system data I/O */
   };
   struct node_info ni;
   int dirty, err;

   /* Check if the inode actually has inline data to convert.
    * f2fs_exist_data checks if the inode has any data blocks allocated,
    * if not, it might mean it's an empty file or something else, but in this context,
    * we are converting *from* inline data, so this check might be redundant or for safety.
    * In reality, for inline conversion, we expect inline data to exist.
    */
   if (!f2fs_exist_data(dn->inode))
   	goto clear_out; /* If no data exists (unexpected in inline conversion), jump to clear_out */

   /* Reserve a new data block for the inode.
    * This is the crucial step to allocate a regular data block to replace inline data.
    * f2fs_reserve_block will handle the block allocation and update dnode accordingly.
    * index 0 is passed, indicating the first block of data (offset 0 in file).
    */
   err = f2fs_reserve_block(dn, 0);/*传进去的是个dn指针
   所以说应该会修改dn的blk_addr为NEW_ADDR*/
   if (err)
   	return err; /* Return error if block reservation fails */

   /* Get node info for the inode's node page.
    * This is likely to get the version number of the node page for concurrency control.
    * 'false' indicates we don't need to lock the node page exclusively at this stage.
    */
   err = f2fs_get_node_info(fio.sbi, dn->nid, &ni, false);
   if (err) {
   	/* If getting node info fails, we need to rollback the block reservation.
   	 * f2fs_truncate_data_blocks_range releases the reserved block (count=1).
   	 */
   	f2fs_truncate_data_blocks_range(dn, 1);
   	f2fs_put_dnode(dn); /* Release the dnode */
   	return err; /* Return error */
   }

   fio.version = ni.version; /* Set the version number in io info */

   /* Sanity check: Ensure that the data block address in dnode is still NEW_ADDR.
    * In f2fs_reserve_block, we expect a new block to be reserved and dn->data_blkaddr
    * to be set to NEW_ADDR initially, which will be replaced with the actual block address
    * during outplace write. If it's not NEW_ADDR, it indicates corruption.
    */
   if (unlikely(dn->data_blkaddr != NEW_ADDR)) {
   	f2fs_put_dnode(dn); /* Release dnode */
   	set_sbi_flag(fio.sbi, SBI_NEED_FSCK); /* Set superblock flag indicating file system check is needed */
   	f2fs_warn(fio.sbi, "%s: corrupted inline inode ino=%lx, i_addr[0]:0x%x, run fsck to fix.",
   		  __func__, dn->inode->i_ino, dn->data_blkaddr); /* Warn about corruption */
   	f2fs_handle_error(fio.sbi, ERROR_INVALID_BLKADDR); /* Handle the error */
   	return -EFSCORRUPTED; /* Return file system corrupted error */
   }

   /* Bug check: Ensure the page is not already under writeback.
    * This is a sanity check to prevent data corruption or race conditions.
    * folio_test_writeback checks if the folio is currently being written back.
    */
   f2fs_bug_on(F2FS_P_SB(page), folio_test_writeback(page_folio(page)));

   /* Read the inline data from the inode page into the data page.
    * This copies the inline data content from the inode's node page to the newly allocated data page.
    * page_folio(page) gets the folio from the page.
    * dn->inode_page is the page containing the inode metadata and inline data.
    */
   f2fs_do_read_inline_data(page_folio(page), dn->inode_page);
   set_page_dirty(page); /* Mark the data page as dirty, indicating it needs to be written back */

   /* Clear dirty state for I/O.
    * clear_page_dirty_for_io clears the PG_dirty flag and returns whether it was dirty.
    * 'dirty' will be non-zero if the page was dirty before clearing.
    */
   dirty = clear_page_dirty_for_io(page);

   /* Write the data page out to disk to persist the converted data.
    * This is the actual write operation to save the inline data to a regular data block.
    */
   set_page_writeback(page); /* Set page writeback flag */
   fio.old_blkaddr = dn->data_blkaddr; /* Set old block address in io info (should be NEW_ADDR) */
   set_inode_flag(dn->inode, FI_HOT_DATA); /* Set inode flag indicating data is hot (frequently accessed) */
   f2fs_outplace_write_data(dn, &fio); /* Perform out-place write of the data page */
   f2fs_wait_on_page_writeback(page, DATA, true, true); /* Wait for writeback completion */
   if (dirty) {
   	inode_dec_dirty_pages(dn->inode); /* Decrement inode's dirty page count */
   	f2fs_remove_dirty_inode(dn->inode); /* Remove inode from dirty inode list */
   }

   /* Mark inode for append write recovery.
    * This flag might be used for crash recovery, indicating that this inode was involved in
    * an inline data conversion and might need special handling during recovery.
    */
   set_inode_flag(dn->inode, FI_APPEND_WRITE);

   /* Clear inline data and inline flag after successful data writeback.
    * f2fs_truncate_inline_inode clears the inline data area in the inode page.
    * clear_page_private_inline clears the flag indicating inline data is present in the inode page.
    */
   f2fs_truncate_inline_inode(dn->inode, dn->inode_page, 0);
   clear_page_private_inline(dn->inode_page);

clear_out:
   stat_dec_inline_inode(dn->inode); /* Decrement inline inode statistics counter */
   clear_inode_flag(dn->inode, FI_INLINE_DATA); /* Clear inode flag indicating inline data is used */
   f2fs_put_dnode(dn); /* Release the dnode */
   return 0; /* Return success */
}

/**
* f2fs_reserve_block - Reserve a data block for a dnode
* @dn: dnode of data
* @index: page index within the file (pgoff_t)
*
* This function reserves a new data block for the given dnode.
* It gets a dnode for the given index and reserves a new block if needed.
*/


/**
* f2fs_reserve_new_block - Reserve one new data block
* @dn: dnode of data
*
* This function reserves a single new data block for the given dnode.
* It's a wrapper around f2fs_reserve_new_blocks for reserving just one block.
* Should keep dn->ofs_in_node unchanged.
*/
int f2fs_reserve_new_block(struct dnode_of_data *dn)
{
   unsigned int ofs_in_node = dn->ofs_in_node; /* Save current offset in node */
   int ret;

   /* Reserve 1 new block using f2fs_reserve_new_blocks */
   ret = f2fs_reserve_new_blocks(dn, 1);
   dn->ofs_in_node = ofs_in_node; /* Restore original offset in node */
   return ret; /* Return reservation result */
}

/**
* f2fs_reserve_new_blocks - Reserve multiple new data blocks
* @dn: dnode of data
* @count: number of blocks to reserve
*
* This function reserves a specified number of new data blocks for the given dnode.
* dn->ofs_in_node will be returned with up-to-date last block pointer.
*/
int f2fs_reserve_new_blocks(struct dnode_of_data *dn, blkcnt_t count)
{
   struct f2fs_sb_info *sbi = F2FS_I_SB(dn->inode); /* Get superblock info */
   int err;

   if (!count)
   	return 0; /* If count is 0, return success immediately */

   /* Check if FI_NO_ALLOC flag is set, indicating no allocation is allowed. */
   if (unlikely(is_inode_flag_set(dn->inode, FI_NO_ALLOC)))
   	return -EPERM; /* Return permission error if no allocation is allowed */

   /* Increment valid block count in superblock and inode.
    * inc_valid_block_count increases the count of valid data blocks and updates metadata.
    * 'true' indicates that this is for reservation and might involve quota checks.
    */
   err = inc_valid_block_count(sbi, dn->inode, &count, true);
   if (unlikely(err))
   	return err; /* Return error if incrementing block count fails */

   trace_f2fs_reserve_new_blocks(dn->inode, dn->nid,
   					dn->ofs_in_node, count); /* Trace block reservation event */

   /* Wait for writeback on the node page before proceeding with allocation.
    * This ensures consistency and avoids race conditions when modifying node page metadata.
    */
   f2fs_wait_on_page_writeback(dn->node_page, NODE, true, true);

   /* Loop to reserve 'count' number of blocks.
    * dn->ofs_in_node is incremented in each iteration to point to the next block entry in the node page.
    */
   for (; count > 0; dn->ofs_in_node++) {
   	block_t blkaddr = f2fs_data_blkaddr(dn); /* Get current data block address from dnode */

   	/* Check if the current block address is NULL_ADDR, meaning it's available for allocation. */
   	if (blkaddr == NULL_ADDR) {
   		__set_data_blkaddr(dn, NEW_ADDR); /* Mark the block address as NEW_ADDR, indicating it's reserved */
   		count--; /* Decrement the remaining block count */
   	}
   	/* If blkaddr is not NULL_ADDR, it means block is already allocated, skip to next entry */
   }

   /* Mark the node page as dirty because we have modified block addresses in it. */
   if (set_page_dirty(dn->node_page))
   	dn->node_changed = true; /* Set dnode's node_changed flag */
   return 0; /* Return success */
}
```