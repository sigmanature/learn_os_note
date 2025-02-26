
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

```C
#define compound_head(page)	((typeof(page))_compound_head(page)):
```
- `#define compound_head(page)	((typeof(page))_compound_head(page))`
- `#define compound_head(page) ...`: 定义一个宏 compound_head。
- `((typeof(page))_compound_head(page))`: 宏展开。
- `_compound_head(page)`: 调用我们刚刚分析的 _compound_head 函数。
- `(typeof(page))`: 获取输入 page 参数的类型。 如果 page 的类型为 struct page *，则 typeof(page) 为 struct page *。
- `((typeof(page)) ...)`: 将 _compound_head(page) 的结果强制转换为与输入 page 相同的类型。 由于 _compound_head 返回 unsigned long，因此此强制转换将其转换回 struct page *（或 const struct page *，具体取决于输入 page 类型）。
compound_head 的目的:

此宏提供了一种类型安全的方式来获取复合页面（或 folio）的头页面。 它本质上包装了 _compound_head 函数，并确保返回类型与输入页面类型匹配。 这对于类型正确性以及避免在不同上下文中使用宏时出现编译器警告或错误非常重要。
