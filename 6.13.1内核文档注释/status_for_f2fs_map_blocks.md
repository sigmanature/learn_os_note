```C
/*
 * State of block returned by f2fs_map_blocks.
 */
#define F2FS_MAP_NEW		(1U << 0)
#define F2FS_MAP_MAPPED		(1U << 1)
#define F2FS_MAP_DELALLOC	(1U << 2)
#define F2FS_MAP_FLAGS		(F2FS_MAP_NEW | F2FS_MAP_MAPPED |\
				F2FS_MAP_DELALLOC)
```
