# AQS
![[Pasted image 20230820200226.png]]
1. 通过一个int同步状态码，和一个队列来控制多个线程访问资源；
	1. 状态值为0（`state=0`）表示没有线程获得锁，通过原子操作修改状态；
	2. 通过双向链表实现FIFO，头节点为哑节点，其他每个节点都会关联一个线程，当上一个资源释放时，按序遍历链表，唤醒线程；
2. 公平/非公平
	1. 非公平：每次都尝试获取锁，失败后直接加入到队列中；
	2. 公平：队列不存在等待的节点时才尝试获取锁；
3. 共享与独占
	1. 独占 只能有一个线程访问
	2. 读写锁
		1.  state 高16为 读锁，低16为写锁；
		2. 写锁为独占锁，有线程获取了写锁的情况，只有同一线程才能获取读锁；
4. 可重入:  `AbstractOwnableSynchronizer#setExclusiveOwnerThread`   与当前获取锁的线程是否相同，相同直接访问
5. CountDownLatch： 计数等待所有线程执行完，也就是state == 0 。
## 参考链接
- [Java并发学习笔记（八）：AQS（AbstractQueuedSynchronizer）、ReentrantLock 原理、读写锁使用和原理\_aqs读写锁原理\_Miracle42的博客-CSDN博客](https://blog.csdn.net/han_zhuang/article/details/106535716)
- [Java并发编程实战: AQS 源码 史上最详尽图解+逐行注释\_51CTO博客\_java aqs源码分析](https://blog.51cto.com/universsky/5898269)
# 线程
1. wait/notify/notifyAll
2. yield
3. join
4. sychronized
5. volatile
6. 生产者消费者模型
	1.  用while不用if的原因：多个消费者可之间可以互相唤醒，导致判断一直成立
7. 死锁
# 网络编程
1. 三次握手
	1. 客户端发起连接请求，发送SYN报文，同步序号为x，进入SYN-SEND状态；
	2. 服务端收到SYN报文，回应一个 SYN 和ACK报文，同步序号为y 确认序号为x+1，进入SYN_RECV
	3. 客户端收到服务端的SYN报文，回应一个ACK报文 确认序号为y+1,进入Established状态
2. 四次挥手
	1.  某个应用进程首先调用close, 执行主动关闭，发送一个FIN报文，进入FIN_WAIT_1状态
	2. 对端收到FIN后，
	3. z
	4. c
3. 为什么要三次握手？
	**为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误。主要防止资源的浪费。**

## 参考连接
1. [TCP是什么？为什么要三次握手四次挥手？ （本文近9千字，建议收藏） - 知乎](https://zhuanlan.zhihu.com/p/128940209)
# 集合
空间与时间的关系
## HashMap
1. 2的幂 让尽可能多的位数参与index的计算
## SparseArray
## ConcurrentHashMap

## 参考链接

- [【Android面试题】2023最新面试专题：网络编程（一） - 掘金](https://juejin.cn/post/7257386139849326653)
- [轻松掌握RecyclerView缓存机制 - 掘金](https://juejin.cn/post/7244452419458777144)
- [Java中的生产者/消费者模型\_java 生产者消费者模型\_青春路上的小蜜蜂的博客-CSDN博客](https://blog.csdn.net/u010257931/article/details/131996016)
- [JVM经典面试题总结(下) - 掘金](https://juejin.cn/post/7268314195299500073)

