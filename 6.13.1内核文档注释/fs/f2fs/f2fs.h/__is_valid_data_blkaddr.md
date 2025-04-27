```C
static inline bool __is_valid_data_blkaddr(block_t blkaddr)
{
	if (blkaddr == NEW_ADDR || blkaddr == NULL_ADDR ||
			blkaddr == COMPRESS_ADDR)
		return false;/*预分配的地址,空洞地址以及压缩地址都视作非有效块,在f2fs_map_blocks中统一视作空地址*/
	return true;
}
```
