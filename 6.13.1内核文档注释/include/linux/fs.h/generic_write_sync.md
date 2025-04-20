```C
/*
 * Sync the bytes written if this was a synchronous write.  Expect ki_pos
 * to already be updated for the write, and will return either the amount
 * of bytes passed in, or an error if syncing the file failed.
 */
static inline ssize_t generic_write_sync(struct kiocb *iocb, ssize_t count)
{
	if (iocb_is_dsync(iocb)) {
		int ret = vfs_fsync_range(iocb->ki_filp,
				iocb->ki_pos - count, iocb->ki_pos - 1,
				(iocb->ki_flags & IOCB_SYNC) ? 0 : 1);
		if (ret)
			return ret;
	} else if (iocb->ki_flags & IOCB_DONTCACHE) {
		struct address_space *mapping = iocb->ki_filp->f_mapping;

		filemap_fdatawrite_range_kick(mapping, iocb->ki_pos - count,
					      iocb->ki_pos - 1);
	}

	return count;
}
```