```C
static inline void f2fs_put_page(struct page *page, int unlock)
{
	if (!page)
		return;

	if (unlock) {
		f2fs_bug_on(F2FS_P_SB(page), !PageLocked(page));
		unlock_page(page);
	}
	put_page(page);
}
```
用一个控制开关unlock。为0表示不解锁直接减少引用计数(必要时释放)。
为1表示解锁释放。 
