```C
struct kiocb {
	struct file		*ki_filp;
	loff_t			ki_pos;/*进行buffer write的起始位置吧*/
	void (*ki_complete)(struct kiocb *iocb, long ret);
	void			*private;
	int			ki_flags;
	u16			ki_ioprio; /* See linux/ioprio.h */
	union {
		/*
		 * Only used for async buffered reads, where it denotes the
		 * page waitqueue associated with completing the read. Valid
		 * IFF IOCB_WAITQ is set.
		 */
		struct wait_page_queue	*ki_waitq;
		/*
		 * Can be used for O_DIRECT IO, where the completion handling
		 * is punted back to the issuer of the IO. May only be set
		 * if IOCB_DIO_CALLER_COMP is set by the issuer, and the issuer
		 * must then check for presence of this handler when ki_complete
		 * is invoked. The data passed in to this handler must be
		 * assigned to ->private when dio_complete is assigned.
		 */
		ssize_t (*dio_complete)(void *data);
	};
};
```