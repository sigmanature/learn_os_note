```C
/**
 * struct readahead_control - Describes a readahead request.
 * readahead_control数据结构,在预读相关函数中的使用啊,实际上很像是把用户对文件读的很多函数参数打包到了这个数据结构之中
 * 同时又和内核中其他和预读相关以及和文件相关的数据结构产生交互。这样既避免了内核中后续处理预读逻辑的函数总是要传递很多参数。
 * A readahead request is for consecutive pages.  Filesystems which
 * implement the ->readahead method should call readahead_page() or
 * readahead_page_batch() in a loop and attempt to start I/O against
 * each page in the request.
 *
 * Most of the fields in this struct are private and should be accessed
 * by the functions below.
 *
 * @file: The file, used primarily by network filesystems for authentication.
 *	  May be NULL if invoked internally by the filesystem.
 * @mapping: Readahead this filesystem object.
 * @ra: File readahead state.  May be NULL.
 */
struct readahead_control {
	struct file *file;
	struct address_space *mapping;
	struct file_ra_state *ra;
/* private: use the readahead_* accessors instead */
	pgoff_t _index;/*预读的首个page的索引*/
	unsigned int _nr_pages;/*预读的总页数*/
	unsigned int _batch_count;
	bool _workingset;
	unsigned long _pflags;
};

#define DEFINE_READAHEAD(ractl, f, r, m, i)				\
	struct readahead_control ractl = {				\
		.file = f,						\
		.mapping = m,						\
		.ra = r,						\
		._index = i,						\
	}
    /*这个宏就是用来初始化一个readahead_control数据结构的。
    最经典的使用是在函数page_cache_sync_readahead之中*/
    /**
 * page_cache_sync_readahead - generic file readahead
 * @mapping: address_space which holds the pagecache and I/O vectors
 * @ra: file_ra_state which holds the readahead state
 * @file: Used by the filesystem for authentication.
 * @index: Index of first page to be read.
 * @req_count: Total number of pages being read by the caller.
 *
 * page_cache_sync_readahead() should be called when a cache miss happened:
 * it will submit the read.  The readahead logic may decide to piggyback more
 * pages onto the read request if access patterns suggest it will improve
 * performance.
 */
static inline
void page_cache_sync_readahead(struct address_space *mapping,
		struct file_ra_state *ra, struct file *file, pgoff_t index,
		unsigned long req_count)
{
	DEFINE_READAHEAD(ractl, file, ra, mapping, index);
	page_cache_sync_ra(&ractl, req_count);
}
/*接下来全是和ractl数据结构相关的一些成员函数*/
/**
 * readahead_length - The number of bytes in this readahead request.
 * @rac: The readahead request.
 */
static inline size_t readahead_length(struct readahead_control *rac)
{
	return rac->_nr_pages * PAGE_SIZE;/*以字节为单位返回预读长度。有个很重要的应用是用来构造iomap数据结构*/
}

/**
 * readahead_index - The index of the first page in this readahead request.
 * @rac: The readahead request.
 */
static inline pgoff_t readahead_index(struct readahead_control *rac)
{
	return rac->_index;
}
/**
 * readahead_pos - The byte offset into the file of this readahead request.
 * @rac: The readahead request.
 */
static inline loff_t readahead_pos(struct readahead_control *rac)
{
	return (loff_t)rac->_index * PAGE_SIZE;/*返回预读控制中的文件位置。实质是将起始页索引以字节为单位返回。比较重要的用途是初始化iomap_iter数据结构*/
}
/**
 * readahead_count - The number of pages in this readahead request.
 * @rac: The readahead request.
 */
static inline unsigned int readahead_count(struct readahead_control *rac)
{
	return rac->_nr_pages;/*以页面数量为单位返回readahead的长度*/
}
```
