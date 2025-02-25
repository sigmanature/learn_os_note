```C
static inline void mapping_set_large_folios(struct address_space *mapping)
{
	mapping_set_folio_order_range(mapping, 0, MAX_PAGECACHE_ORDER);
}
static inline void mapping_set_folio_order_range(struct address_space *mapping,
						 unsigned int min,
						 unsigned int max)
{
	if (!IS_ENABLED(CONFIG_TRANSPARENT_HUGEPAGE))
		return;

	if (min > MAX_PAGECACHE_ORDER)
		min = MAX_PAGECACHE_ORDER;

	if (max > MAX_PAGECACHE_ORDER)
		max = MAX_PAGECACHE_ORDER;

	if (max < min)
		max = min;

	mapping->flags = (mapping->flags & ~AS_FOLIO_ORDER_MASK) |
		(min << AS_FOLIO_ORDER_MIN) | (max << AS_FOLIO_ORDER_MAX);
}
```
AS_FOLIO_ORDER_MIN和AS_FOLIO_ORDER_MAX是定义在pagemap.h中enum mapping_flags中的标志位
前者是16后者是21。内核注释中提到address_space的flags中16到25位用于标志folios的order。
'mapping_set_large_folios'由于调用' mapping_set_folio_order_range '中传的min是0因此
'(min << AS_FOLIO_ORDER_MIN)'直接就是0。然后'(max << AS_FOLIO_ORDER_MAX)'
max的值是为'MAX_PAGECACHE_ORDER'
其定义是
```C
#define MAX_PAGECACHE_ORDER	min(MAX_XAS_ORDER, PREFERRED_MAX_PAGECACHE_ORDER)
```
这个值不是7就是8。' (max << AS_FOLIO_ORDER_MAX) '刚好就占了25位。
'(mapping->flags & ~AS_FOLIO_ORDER_MASK) |
		(min << AS_FOLIO_ORDER_MIN) | (max << AS_FOLIO_ORDER_MAX)'最终的值实际上就是
flags的16到25位只设置了最大的order。比如8是1000 左移21位就是1后面24个0
但是其他地方是怎么和这个范围的比特位交互的,我们该具体怎么解读这些bit的作用还需联系其他代码。
