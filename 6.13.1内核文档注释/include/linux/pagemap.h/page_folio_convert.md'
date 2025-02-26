```C
static __always_inline unsigned long _compound_head(const struct page *page)
{
	unsigned long head = READ_ONCE(page->compound_head);

	if (unlikely(head & 1))
		return head - 1;
	return (unsigned long)page_fixed_fake_head(page);
}
```
在普通的情况下`page_fixed_fake_head(page)`直接什么都不做返回page本身。<br>
