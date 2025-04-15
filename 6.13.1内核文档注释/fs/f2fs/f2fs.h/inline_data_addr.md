**相关函数**
* [get_dnode_addr](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/f2fs.h/f2fs_datablk_addr.md)
```C
static inline void *inline_data_addr(struct inode *inode, struct page *page)
{
	__le32 *addr = get_dnode_addr(inode, page);

	return (void *)(addr + DEF_INLINE_RESERVED_SIZE);
}
```
这里我们回顾一下`get_dnode_addr`的定义: