data.c文件中包含f2fs和块设备真正进行io的重要核心函数,也是整f2fs所有和pagecache进行io操作必须调用的函数
`f2fs_submit_page_bio`
另外,还包含了对于普通io来说至关重要的从内存页面到真正物理块映射关系的函数`f2fs_map_blocks`
__bio_alloc调用关系流程图
```mermaid
graph LR
A['__bio_alloc']-->B['bio_alloc_bioset']-->C{是读io吗?}
C--是-->D[将bi_end_io设置为f2fs_read_end_io]
C--否-->E[将bi_end_io设置为f2fs_write_end_io]
D-->继续
E-->继续-->F{是否有wbc控制块?}--是-->G[用wbc控制块初始化bio]
```