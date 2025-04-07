```C
static inline sector_t iomap_sector(const struct iomap *iomap, loff_t pos)
{
	return (iomap->addr + pos - iomap->offset) >> SECTOR_SHIFT;
}
```
整个iomap的灵魂函数。
iomap_sector在iomap_readpage_iter中被调用,其参数pos代表的是从iomap_readahead_iter中传过来的done加上iter->pos算出来的。done的取值区间是`[0,iomap_length(iter)-1]`。展开计算也就是
`[0,min(iter->length,iomap->offset+iomap->length-iter->pos)]`iomap->addr是之前得到的逻辑扇区映射区间的起始字节地址(也就是操作系统层面的"物理地址")。iomap->offset是以文件系统块对齐过的文件逻辑偏移量。实际上就是iter->pos取整除文件系统块算出来的。简单起见,假设`min(iter->length,iomap->offset+iomap->length-iter->pos)`=`iomap->length-diff`。其中`diff=iter->pos-iomap->offset`。这个值应该就是iter->pos中对文件系统块的余数。 那pos=iter->pos+done的取值区间就是`[iter->pos,iter->pos+iomap->length-diff]`那pos-iomap->offset的取值区间就是`[iter->pos-iomap->offset,diff+iomap->length-diff]`=`[diff,iomap->length]`。看这个区间,神奇的事情发生了。这个算出来的恰恰好好就是精准的从当前逻辑扇区开始,偏移了用户指定要读的偏移量。
