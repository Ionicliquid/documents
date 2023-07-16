# Binder的内存管理
## mmap
将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址与进程虚拟地址空间的一段地址的一一对应关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存。
## Binder中的mmap
### 一次拷贝
Binder 驱动中，服务端进程会事先调用 mmap 将服务端的用户空间的一段虚拟内存和内核空间的一段虚拟内存，映射到同一块物理内存上。这段物理内存，将作为一个缓冲区，接收来自不同客户端的数据。比如，客户端发送事务数据时，内核会通过 copy_from_user 将数据放进缓冲区，然后通知服务端。服务端只需要通过用户空间相应的虚拟地址，就能直接访问数据：
![[Binder的一次拷贝.webp]]
## 参考链接
[mmap实现原理解析](https://blog.csdn.net/weixin_42937012/article/details/121269058)
[图解 Binder：内存管理 - 掘金](https://juejin.cn/post/7244734179828203579)
[[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)j](https://www.cnblogs.com/huxiao-tee/p/4660352.html)![[Binder的一次拷贝.webp]]