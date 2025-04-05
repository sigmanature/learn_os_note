Linux在io之中进行预读的所有函数。最常见的使用场景就是在buffered read中的filemap_read函数中被使用。是Linux进行预读的传统方案。<br>
[filemap_read](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c)
以同步预读page_readahead_sync为例子,看一下它们的函数调用关系图。
```mermaid
graph LR
    subgraph a[page_cache_sync_readahead]
    A[初始化readahead_control]-->B[page_cache_sync_ra]
    end
B-->C{如果可能是随机读?}
C--是-->E[do_page_cache_ra]-->G[page_cache_ra_unbounded]
C--否-->F[page_cache_ra_order]-->H{不支持大页面?}--是-->E
H--否-->I[自适应预读阶数分配]-->J[read_pages]
click A "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/include/linux/pagemap.h/readahead_control.md"
click E "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click F "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/page_cache_ra_order.md"
click B "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click G "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/readahead_funcs.md"
click J "https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/read_pages.md"
```