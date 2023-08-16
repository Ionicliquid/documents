# JVM的内存管理
JVM是一种规范，不同的虚拟机参考规范来实现。
运行时数据区：Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。主要分为堆、程序计数器、方法区、虚拟机栈和本地方法栈。
![[JVM 运行时数据区.png]]
### 虚拟机栈
栈是由一系列帧(Frame)组成(因此Java栈也叫作帧栈)，是线程私有的，存储当前线程运行过程所需的数据、指令返回地址。
#### 栈帧
每个方法的执行，都有一个栈帧入栈，栈帧分为四大区域：
1. 局部变量表：主要存储方法中定义的局部变量，它是一个数组，除了存储局部变量之外，还会存储方法的参数；
2.  操作数栈：主要用于在方法的执行过程中，根据字节码指令，往栈内压入数据或者提取数据。主要的操作像赋值、交换、四大运算（+ - x ÷）等；
3. 返回地址：正常返回（调用程序计数器中的地址作为返回）、异常的话（通过异常处理器表<非栈帧中的>来确定）；
4. 动态链接；
### 本地方法栈
### 程序计数器
当前线程执行的字节码的行号指示器；各线程之间独立存储，互不影响。程序计数器是JVM中唯一不会OOM（OutOfMemory）的内存区域。
### 方法区
方法区是线程共享的，通常用来保存装载的类的结构信息。 通常和元空间关联在一起，但具体的跟JVM实现和版本有关。
#### 运行时常量池
 是Class文件中每个类或接口的常量池表，在运行期间的表示形式，通常包括：类的版本、字段、方法、接口等信息。运行时常量池在方法区中分配 通常在加载类和接口到JVM后，就创建相应的运行时常量池。
###  堆
堆是 JVM 上最大的内存区域，我们申请的几乎所有的对象，都是在这里存储的。我们常说的垃圾回收，操作的对象就是堆。
### 直接内存
## 对象的分配及垃圾回收机制
### 对象的创建过程
1. 检查加载：检查类是否被加载；
2. 分配内存：
	1. Java堆规整采用指针碰撞，否则空闲列表；
	2. CAS 或者本地线程缓存（TLAB， 每个线程预先分配一小块内存）来保证并发下的线程安全；
3. 内存空间初始化： 将分配到的内存空间（不包括对象头）都初始化为零值，保证对象的实例字段在Java代码中可以不赋初始值就直接使用，使程序能访问到这些字段的数据类型所对应的零值。
4. 设置：设置对象头信息：如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码（实际上对象的哈希码会延后到真正调用Object::hashCode()方法时才计算）、对象的GC分代年龄等信息。
5. 对象初始化：执行构造方法；
### 对象的内存布局
1. 对象头
2. 实例数据
3. 对齐填充（非必须）
### 分代收集理论
- 新生代收集（Minor GC/Young GC）：指目标只是新生代的垃圾收集；
- 整堆收集（Full GC）：收集整个Java堆和方法区的垃圾收集
### 对象的分配策略
![[对象分配.png]]
1. 栈上分配
2.  对象优先在Eden分配
3. 大对象直接进入老年代
### 对象的回收策略
1. 长期存活的对象将进入老年代：每个对象定义了一个对象年龄（Age）计数器，存储在对象头中，经过Minor GC后仍然存活，并且能被Survivor容纳的话，该对象会被移动到Survivor空间中，并且将其对象年龄设为1岁。
2. 动态对象年龄判定： 不一定要达到阈值年龄（15）才会进入老年代；
3. 空间分配担保：Minor GC之前，需要老年代去历史平均值来担保还有足够的空间能够接受新生代存活对象，避免频繁的触发Full GC;
### 判断对象的存活
#### 引用计数法
> 在对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加一；当引用失效时，计数器值就减一；任何时刻计数器为零的对象就是不可能再被使用的。
~~**无法解决循环引用的问题**~~
#### 可达性分析算法
> 通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。
![[gc_roots.png]]
1. <font color="#ff0000">局部变量</font>：在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。
2. <font color="#ff0000">静态变量</font>：在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量。
3. <font color="#ff0000">常量池</font>：在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引
4. <font color="#ff0000">JNI（指针</font>）：在本地方法栈中JNI（即通常所说的Native方法）引用的对象。
5. Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器。
6. 所有被同步锁（synchronized关键字）持有的对象。
7. 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。
#### 引用的分类
-  <font color="#ff0000"><font color="#ff0000">强引用</font>（Object obj=new Object()）</font>：无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象；
- <font color="#ff0000">软引用</font><font color="#ff0000">（SoftReference）</font>：软引用是用来描述一些还有用，但非必须的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。
- <font color="#ff0000">弱引用（WeakReference）</font>：弱引用也是用来描述那些非必须对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。
- <font color="#ff0000">虚引用（PhantomReference）</font>：也称为“幽灵引用”或者“幻影引用”，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。
### 垃圾回收算法
#### 标记-清除算法
![[标记清除.png]]
缺点：
1. 执行效率不稳定：如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低；
2. 内存碎片化：标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作；
#### 标记-复制算法
![[标记复制算法.png]]
可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。
优点：
1. 解决清除算法的效率低问题；
2. 没有内存碎片；
缺点：
1. 空间浪费：将可用内存缩小为了原来的一半；
#### 标记整理算法
标记-复制算法针对对象存活率高的情况效率低下，所有针对老年代一般采用标记整理算法；标记-清除算法与标记-整理算法的本质差异在于前者是一种非移动式的回收算法，而后者是移动式的。是否移动回收后的存活对象是一项优缺点并存的风险决策：
>首先标记出所有需要回收的对象，在标记完成后，后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。标记整理算法虽然没有内存碎片，但是效率偏低。我们看到标记整理与标记清除算法的区别主要在于对象的移动。对象移动不单单会加重系统负担，同时需要<font color="#ff0000">全程暂停用户线程</font>才能进行，同时所有引用对象的地方都需要更新（直接指针需要调整）。
![[标记整理.png]]
特点：
1. 没有内存碎片；
2. 指针需要移动；
# 线程池
### 任务的执行流程
#### execute
```java
public void execute(Runnable command) {  
    if (command == null)  
        throw new NullPointerException();  
	 int c = ctl.get();  
    if (workerCountOf(c) < corePoolSize) {  //1
        if (addWorker(command, true))  
            return;  
        c = ctl.get();  
    }  
    if (isRunning(c) && workQueue.offer(command)) {  
        int recheck = ctl.get();  
        if (! isRunning(recheck) && remove(command))  
            reject(command);  
        else if (workerCountOf(recheck) == 0)  
            addWorker(null, false);  
    }  
    else if (!addWorker(command, false))  
        reject(command);  
}
```
源码中也有对流程的注释，执行过程分为三种情况。
1. 运行线程数小于核心线程池，直接新建线程执行；
2.  尝试加入到阻塞队列，如果成功，测
3. 直接加入到任务队列，
#### addWorker：检查线程池状态 创建并启动线程；
``` java
private boolean addWorker(Runnable firstTask, boolean core) {  
    retry:  
    for (int c = ctl.get();;) {  
       // 线程状态state != RUNNING, 
        if (runStateAtLeast(c, SHUTDOWN)  
            && (runStateAtLeast(c, STOP)  // state>=STOP
                || firstTask != null  
                || workQueue.isEmpty()))  
            return false;  
  
        for (;;) {  
            if (workerCountOf(c)  
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))  //如果为核心线程，不大于核心线程数，非核心线程不大于最大线程数
                return false;  
            if (compareAndIncrementWorkerCount(c))  // CAS 操作增加有效线程数
                break retry;  
            c = ctl.get();  // Re-read ctl  
            if (runStateAtLeast(c, SHUTDOWN))  
                continue retry;  
            // else CAS failed due to workerCount change; retry inner loop  
        }  
    }  
  
    boolean workerStarted = false;  
    boolean workerAdded = false;  
    Worker w = null;  
    try {  
        w = new Worker(firstTask);  // 创建线程
        final Thread t = w.thread;  
        if (t != null) {  
            final ReentrantLock mainLock = this.mainLock;  
            mainLock.lock();  
            try {  
                int c = ctl.get();  
  
                if (isRunning(c) ||  
                    (runStateLessThan(c, STOP) && firstTask == null)) {  
                    if (t.getState() != Thread.State.NEW)  
                        throw new IllegalThreadStateException();  
                    workers.add(w);  
                    workerAdded = true;  
                    int s = workers.size();  
                    if (s > largestPoolSize)  
                        largestPoolSize = s;  
                }  
            } finally {  
                mainLock.unlock();  
            }  
            if (workerAdded) {  
                container.start(t);  //启动线程
                workerStarted = true;  
            }  
        }  
    } finally {  
        if (! workerStarted)  
            addWorkerFailed(w);  
    }  
    return workerStarted;  
}
```
runWorker:执行任务
``` java
final void runWorker(Worker w) {  
    Thread wt = Thread.currentThread();  
    Runnable task = w.firstTask;  
    w.firstTask = null;  
    w.unlock(); // allow interrupts  
    boolean completedAbruptly = true;  
    try {  
        while (task != null || (task = getTask()) != null) {  
            w.lock();  
	         if ((runStateAtLeast(ctl.get(), STOP) ||  
                 (Thread.interrupted() &&  
                  runStateAtLeast(ctl.get(), STOP))) &&  
                !wt.isInterrupted())  
                wt.interrupt();  
            try {  
                beforeExecute(wt, task);  
                try {  
                    task.run();  
                    afterExecute(task, null);  
                } catch (Throwable ex) {  
                    afterExecute(task, ex);  
                    throw ex;  
                }  
            } finally {  
                task = null;  
                w.completedTasks++;  
                w.unlock();  
            }  
        }  
        completedAbruptly = false;  
    } finally {  
        processWorkerExit(w, completedAbruptly);  
    }  
}

private Runnable getTask() {  
    boolean timedOut = false; // Did the last poll() time out?  
  
    for (;;) {  
        int c = ctl.get();  
  
        // Check if queue empty only if necessary.  
        if (runStateAtLeast(c, SHUTDOWN)  
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {  
            decrementWorkerCount();  
            return null;        }  
  
        int wc = workerCountOf(c);  
  
        // Are workers subject to culling?  
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;  // 允许核心线程超时 或者 当前线程数大于核心线程
  
        if ((wc > maximumPoolSize || (timed && timedOut))  
            && (wc > 1 || workQueue.isEmpty())) {  
            if (compareAndDecrementWorkerCount(c))  
                return null;  
            continue;        }  
  
        try {  
            Runnable r = timed ?  
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  
                workQueue.take();  
            if (r != null)  
                return r;  
            timedOut = true;  
        } catch (InterruptedException retry) {  
            timedOut = false;  
        }  
    }  
}
```

# AQS
1. 通过一个int同步状态码，和一个队列来控制多个线程访问资源；
2. 状态值为0（`state=0`）表示没有线程获得锁；
3. 通过双向链表实现FIFO，头节点为哑节点，其他每个节点都会关联一个线程，当上一个资源释放时，按序遍历链表，唤醒线程；
**park与unpark的实现原理**
## ReentrantLock
### 公平/非公平
``` java
//FairSync
protected final boolean tryAcquire(int acquires) {  
    final Thread current = Thread.currentThread();  
    int c = getState();  
    if (c == 0) {  
        if (!hasQueuedPredecessors() &&  
            compareAndSetState(0, acquires)) {  
            setExclusiveOwnerThread(current);  
            return true;        }  
    }  
    else if (current == getExclusiveOwnerThread()) {  
        int nextc = c + acquires;  
        if (nextc < 0)  
            throw new Error("Maximum lock count exceeded");  
        setState(nextc);  
        return true;    }  
    return false;  
}
//Sync
final boolean nonfairTryAcquire(int acquires) {  
    final Thread current = Thread.currentThread();  
    int c = getState();  
    if (c == 0) {  
        if (compareAndSetState(0, acquires)) {  
            setExclusiveOwnerThread(current);  
            return true;        }  
    }  
    else if (current == getExclusiveOwnerThread()) {  
        int nextc = c + acquires;  
        if (nextc < 0) // overflow  
            throw new Error("Maximum lock count exceeded");  
        setState(nextc);  
        return true;    }  
    return false;  
}
//AbstractQueuedSynchronizer
public final boolean hasQueuedPredecessors() {  
    Node h, s;  
    if ((h = head) != null) {  
        if ((s = h.next) == null || s.waitStatus > 0) {  
            s = null; // traverse in case of concurrent cancellation  
            for (Node p = tail; p != h && p != null; p = p.prev) {  
                if (p.waitStatus <= 0)  
                    s = p;  
            }  
        }  
        if (s != null && s.thread != Thread.currentThread())  
            return true;  
    }  
    return false;  
}
```
1. 非公平：每次都尝试获取锁，失败后直接加入到队列中；
2. 公平：队列不存在等待的节点时才尝试获取锁；
## 参考链接
- [Java并发学习笔记（八）：AQS（AbstractQueuedSynchronizer）、ReentrantLock 原理、读写锁使用和原理\_aqs读写锁原理\_Miracle42的博客-CSDN博客](https://blog.csdn.net/han_zhuang/article/details/106535716)
# 线程

# 集合
## HashMap