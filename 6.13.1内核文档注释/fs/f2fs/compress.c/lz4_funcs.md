```C
#ifdef CONFIG_F2FS_FS_LZ4 // Only compile this code if LZ4 compression is enabled in the kernel config

// Function to initialize the compression context for LZ4/LZ4HC
static int lz4_init_compress_ctx(struct compress_ctx *cc)
{
	// Default size needed for LZ4 compression workspace
	unsigned int size = LZ4_MEM_COMPRESS;

#ifdef CONFIG_F2FS_FS_LZ4HC // If LZ4 High Compression is also enabled
	// Check the compression level set for the specific inode
	if (F2FS_I(cc->inode)->i_compress_level)
		// If a non-zero level is set (meaning HC), use the larger HC workspace size
		size = LZ4HC_MEM_COMPRESS;
#endif

	// Allocate memory for the compression workspace (private data for LZ4)
	// f2fs_kvmalloc is a wrapper around kvmalloc, allocating from slab or vmalloc
	// GFP_NOFS prevents filesystem recursion during memory allocation
	cc->private = f2fs_kvmalloc(F2FS_I_SB(cc->inode), size, GFP_NOFS);
	if (!cc->private) // Check if allocation failed
		return -ENOMEM; // Return memory allocation error

	/*
	 * we do not change cc->clen to LZ4_compressBound(inputsize) to
	 * adapt worst compress case, because lz4 compressor can handle
	 * output budget properly.
	 * Note: This comment explains why cc->clen isn't set to the theoretical maximum.
	 * LZ4 functions can take the actual output buffer size (cc->clen) and will
	 * stop if the compressed data exceeds it, returning an error or the size used.
	 */
	// Set the initial compressed length budget (clen).
	// It's the original data length (rlen, typically cluster size in bytes)
	// minus one page (reserved, possibly for metadata or worst-case expansion?)
	// minus the size of the F2FS compression header.
	// This is the maximum space the compressed data can occupy within the allocated blocks.
	cc->clen = cc->rlen - PAGE_SIZE - COMPRESS_HEADER_SIZE;
	return 0; // Initialization successful
}

// Function to destroy the LZ4/LZ4HC compression context
static void lz4_destroy_compress_ctx(struct compress_ctx *cc)
{
	// Free the allocated workspace memory
	kvfree(cc->private);/*把压缩自己private中的上下文释放掉了*/
	// Set the pointer to NULL to avoid dangling pointers
	cc->private = NULL;
}

// Function to perform LZ4/LZ4HC compression on pages gathered in the context
static int lz4_compress_pages(struct compress_ctx *cc)
{
	int len = -EINVAL; // Initialize length to an error value
	// Get the compression level specified for this inode (0 for default LZ4, >0 for LZ4HC)
	unsigned char level = F2FS_I(cc->inode)->i_compress_level;

	if (!level) // If level is 0, use standard LZ4
		// Call the standard LZ4 compression function
		// cc->rbuf: input buffer (concatenated pages)
		// cc->cbuf->cdata: output buffer (for compressed data)
		// cc->rlen: input length (total size of pages in bytes)
		// cc->clen: output buffer size limit
		// cc->private: LZ4 workspace memory
		len = LZ4_compress_default(cc->rbuf, cc->cbuf->cdata, cc->rlen,
						cc->clen, cc->private);
#ifdef CONFIG_F2FS_FS_LZ4HC // If LZ4HC is enabled
	else // If level is non-zero, use LZ4 High Compression
		// Call the LZ4HC compression function
		// level: desired HC compression level
		len = LZ4_compress_HC(cc->rbuf, cc->cbuf->cdata, cc->rlen,
					cc->clen, level, cc->private);
#endif
	if (len < 0) // Check if compression returned an error
		return len; // Propagate the error
	if (!len) // Check if compression resulted in zero output length (might indicate incompressible data or error)
		return -EAGAIN; // Return EAGAIN, often used to signal "try again" or "operation cannot be completed now" (here maybe means "write uncompressed")

	// Compression successful, update clen to the actual compressed size
	cc->clen = len;
	return 0; // Return success
}

// Function to perform LZ4 decompression
static int lz4_decompress_pages(struct decompress_io_ctx *dic)
{
	int ret; // Variable to store the return value of the decompression function

	// Call the safe LZ4 decompression function (protects against buffer overflows)
	// dic->cbuf->cdata: input buffer (compressed data read from disk)
	// dic->rbuf: output buffer (where decompressed pages will be written)
	// dic->clen: input length (size of compressed data)
	// dic->rlen: output buffer size limit (expected decompressed size, cluster size in bytes)
	ret = LZ4_decompress_safe(dic->cbuf->cdata, dic->rbuf,
						dic->clen, dic->rlen);
	if (ret < 0) { // Check if decompression returned an error
		// Log a rate-limited error message
		f2fs_err_ratelimited(F2FS_I_SB(dic->inode),
				"lz4 decompress failed, ret:%d", ret);
		return -EIO; // Return I/O error
	}

	// Calculate the expected decompressed size (number of pages in cluster * page size)
	// dic->log_cluster_size is log2(number of pages per cluster)
	// PAGE_SIZE << dic->log_cluster_size is equivalent to PAGE_SIZE * (2 ^ dic->log_cluster_size)
	if (ret != PAGE_SIZE << dic->log_cluster_size) {
		// Check if the actual decompressed size matches the expected size
		// Log a rate-limited error message if mismatch occurs (indicates corruption)
		f2fs_err_ratelimited(F2FS_I_SB(dic->inode),
				"lz4 invalid ret:%d, expected:%lu",
				ret, PAGE_SIZE << dic->log_cluster_size);
		return -EIO; // Return I/O error
	}
	return 0; // Decompression successful
}

// Function to check if a given compression level is valid for LZ4/LZ4HC
static bool lz4_is_level_valid(int lvl)
{
#ifdef CONFIG_F2FS_FS_LZ4HC // If LZ4HC is enabled
	// Level 0 (default LZ4) is valid, or levels within the valid HC range
	return !lvl || (lvl >= LZ4HC_MIN_CLEVEL && lvl <= LZ4HC_MAX_CLEVEL);
#else // If only standard LZ4 is enabled
	// Only level 0 is valid
	return lvl == 0;
#endif
}

// Define the f2fs_compress_ops structure for LZ4
// This structure holds pointers to the LZ4-specific functions,
// allowing the generic F2FS compression code to call them polymorphically.
static const struct f2fs_compress_ops f2fs_lz4_ops = {
	.init_compress_ctx	= lz4_init_compress_ctx,    // Pointer to LZ4 init function
	.destroy_compress_ctx	= lz4_destroy_compress_ctx, // Pointer to LZ4 destroy function
	.compress_pages		= lz4_compress_pages,       // Pointer to LZ4 compress function
	.decompress_pages	= lz4_decompress_pages,     // Pointer to LZ4 decompress function
	.is_level_valid		= lz4_is_level_valid,       // Pointer to LZ4 level validation function
};
#endif // CONFIG_F2FS_FS_LZ4
```