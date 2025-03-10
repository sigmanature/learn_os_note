``` C
static inline bool folio_test_uptodate(const struct folio *folio)
{
	bool ret = test_bit(PG_uptodate, const_folio_flags(folio, 0));
	/*
	 * Must ensure that the data we read out of the folio is loaded
	 * _after_ we've loaded folio->flags to check the uptodate bit.
	 * We can skip the barrier if the folio is not uptodate, because
	 * we wouldn't be reading anything from it.
	 *
	 * See folio_mark_uptodate() for the other side of the story.
	 */
	if (ret)
		smp_rmb();

	return ret;
}

static inline bool PageUptodate(const struct page *page)
{
	return folio_test_uptodate(page_folio(page));
}
```
内核中用于检查一个headpage或者一个folio的PG_update的标志位是否被设置的函数。
我们最重要的是要理解为什么有PG_update这个标志。
PG_uptodate 是 struct page 结构体中的一个标志位，用于指示 页面数据是否有效且已从存储设备成功读取到内存。
Page Cache 中的页面可以处于多种状态，其中一些关键状态包括：
Not-uptodate (PG_uptodate 未设置): 页面数据 无效或正在读取中。 可能的情况包括：
* 页面刚被分配，尚未从磁盘读取数据。
* 页面读取操作正在进行中，但尚未完成。
* 页面读取操作失败，数据损坏。
* Uptodate (PG_uptodate 设置): 页面数据 有效且已从存储设备成功读取到内存。 表示页面内容是可靠的，可以安全使用。
然后我们补充几种状态:
* Dirty (PG_dirty 设置): 页面内容 已被修改，但尚未写回存储设备。 表示页面内容与存储设备上的数据不一致，需要写回以持久化更改。
* Writeback (PG_writeback 设置): 页面 正在写回存储设备。
有过LRU页面缓存编写经验的话就知道,有这个表示数据是否有效的标志,使得我们在并发的时候能使用更细粒度的锁,io的整个过程,页面自己是要被保护,但是读取页面的
这个进程,就能使用细粒度锁了。所有要拿这个页面数据的进程只需看到有效位被设置即可。当然在Linux里这个表示数据有效的uptodate有更多其他的用途。
