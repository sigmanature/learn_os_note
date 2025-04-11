```C
/*
 * Flags for iomap_begin / iomap_end.  No flag implies a read.
 * 全都是只和写相关的标志位 没有和读相关的。
 */
#define IOMAP_WRITE		(1 << 0) /* writing, must allocate blocks */
#define IOMAP_ZERO		(1 << 1) /* zeroing operation, may skip holes */
#define IOMAP_REPORT		(1 << 2) /* report extent status, e.g. FIEMAP */
#define IOMAP_FAULT		(1 << 3) /* mapping for page fault */
#define IOMAP_DIRECT		(1 << 4) /* direct I/O */
#define IOMAP_NOWAIT		(1 << 5) /* do not block */
#define IOMAP_OVERWRITE_ONLY	(1 << 6) /* only pure overwrites allowed */
#define IOMAP_UNSHARE		(1 << 7) /* unshare_file_range */
#ifdef CONFIG_FS_DAX
#define IOMAP_DAX		(1 << 8) /* DAX mapping */
#else
#define IOMAP_DAX		0
#endif /* CONFIG_FS_DAX */
#define IOMAP_ATOMIC		(1 << 9) /*那只能是原子写了*/
```