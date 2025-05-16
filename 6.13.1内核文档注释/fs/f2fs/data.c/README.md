data.c文件中包含f2fs和块设备真正进行io的重要核心函数,也是整f2fs所有和pagecache进行io操作必须调用的函数
`f2fs_submit_page_bio`
另外,还包含了对于普通io来说至关重要的从内存页面到真正物理块映射关系的函数`f2fs_map_blocks`
__bio_alloc调用关系流程图
```mermaid
graph LR
A['__bio_alloc']-->B['bio_alloc_bioset']-->C{是读io吗?}
C--是-->D[将bi_end_io设置为<br>f2fs_read_end_io]
C--否-->E[将bi_end_io设置为<br>f2fs_write_end_io]
D-->继续
E-->继续-->F{是否有wbc控制块?}--是-->G[用wbc控制块初始化bio]
```
---
这个板块我们仔细捋一下f2fs提交了一个页面io的全部总流程。
```mermaid
graph TD
subgraph a[f2fs_write_single_data_page]
A[构造fio]-->B[压缩块得到LOCK_DONE，
普通则是LOCK_RETRY]-->b
end
subgraph b[f2fs_do_write_data_page]
C[根据是否是原子文件,
设置不同inode的dn]-->D[从inode或cow_inode
找blk_adddr]-->E[如果是inplace更新?]
E--是-->启动folio_writeback-->放掉cp锁-->F[f2fs_inplace_write_data
并返回]
E--否-->G[以LOCK_RETRY方式
try_lockop]-->H[f2fs_outplace_write_data]-->清除掉原子写标志
end
```
```mermaid
graph TD
subgraph a[f2fs_outplace_write_data]
更新age_extent_cache-->构造summary-->A[do_write_page]
end
subgraph b[do_write_page]

end
```