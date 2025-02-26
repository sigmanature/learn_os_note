```C
static __always_inline unsigned long _compound_head(const struct page *page)
{
	unsigned long head = READ_ONCE(page->compound_head);

	if (unlikely(head & 1))
		return head - 1;
	return (unsigned long)page_fixed_fake_head(page);
}
```
函数 _compound_head可以接受head page或者tail page。<br>
在普通的情况下`page_fixed_fake_head(page)`直接什么都不做返回page本身。<br>
因此如果传进去的就是head page那么直接返回自己。(但是地址被以unsigned long的形式返回回来了)。
如果不是的话结果会直接返回page->compound_head-1。那么因此有理由推测page->compound_head字段应该是tail page用来保存head page地址但是最后一个bit始终多了1。
