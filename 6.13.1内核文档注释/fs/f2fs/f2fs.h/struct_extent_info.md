```C
struct extent_info {
	unsigned int fofs;		/* start offset in a file */
	unsigned int len;		/* length of the extent */
	union {
		/* read extent_cache */
		struct {
			/* start block address of the extent */
			block_t blk;
#ifdef CONFIG_F2FS_FS_COMPRESSION
			/* physical extent length of compressed blocks */
			unsigned int c_len;
#endif
		};
		/* block age extent_cache */
		struct {
			/* block age of the extent */
			unsigned long long age;
			/* last total blocks allocated */
			unsigned long long last_blocks;
		};
	};
};
```