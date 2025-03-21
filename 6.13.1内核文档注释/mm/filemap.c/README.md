# filemap.c文件大体介绍
filemap是linux各个文件系统进行io的传统方案。
这个文件里存放的是很多和页面缓存的操作以及进行向文件系统进行页面写入操作的文件。(可能从名字不是很大能看出来)
Linux的io模型的两大重要写入函数:
```C++
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
//通用的写盘函数,经过页面缓冲层
ssize_t generic_file_direct_write(struct kiocb *iocb, struct iov_iter *from)
//通用的写盘函数,跨过缓冲层直接写入磁盘
```
定义于此。
然后,还有各个文件系统最重要的实际执行读io的函数,filemap_read。这个函数的调用链很复杂。详见下图:
```mermaid
graph LR
A[filemap_read]
    click A "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_read.md"
A-->B[filemap_get_pages]
    click B "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_get_pages"
B-->C[filemap_get_read_batch]
B-->D{"folio_batch_count(fbatch)"}
D--为0-->E[page_cache_sync_readahead]-->F[filemap_get_read_batch]
    click E "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
    click F "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_get_read_batch.md"
B-->G{再次检查folio_batch_count?}
G--为0-->H[filemap_create_folio]
```
```mermaid
graph LR
A[page_cache_sync_readahead]-->B[page_cache_sync_ra]-->C{如果可能是随机读?}
C--是-->E[do_page_cache_ra]-->G[page_cache_ra_unbounded]
C--否-->F[page_cache_ra_order]-->H{不支持大页面?}--是-->E
H--否-->I[自适应预读阶数分配]-->J[read_pages]
click E "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click F "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/page_cache_ra_order.md"
click B "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click G "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click J "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/read_pages.md"
```
read_pages函数是真正执行所有预读逻辑的函数。它会调用特定文件系统的预读接口。这里我们以f2fs为例子看一下调用流程图:
```mermaid
graph LR
A[read_pages]-->B[f2fs_readahead]
click B "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_readahead.md"
```