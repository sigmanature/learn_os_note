```C
static inline sector_t iomap_sector(const struct iomap *iomap, loff_t pos)
{
	return (iomap->addr + pos - iomap->offset) >> SECTOR_SHIFT;
}
```
整个iomap的灵魂函数。