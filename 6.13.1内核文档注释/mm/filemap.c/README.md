# filemap.c文件大体介绍
比较令人惊讶的一点是,这个文件里存放的是很多和页面缓存的操作以及进行向文件系统进行页面写入操作的文件。(可能从名字不是很大能看出来)
Linux的io模型的两大重要写入函数:
```C++
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
//通用的写盘函数,经过页面缓冲层
ssize_t generic_file_direct_write(struct kiocb *iocb, struct iov_iter *from)
//通用的写盘函数,跨过缓冲层直接写入磁盘
```
定义于此。除此之外还有大量和folio有关的函数。
