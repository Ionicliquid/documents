Binder是基于内存映射mmap实现的一种进程间的通信方式，
# Binder的内存管理
## mmap
将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址与进程虚拟地址空间的一段地址的一一对应关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存。
## Binder中的mmap
### Binder 进程通信一次拷贝的秘密
Binder 驱动中，服务端进程会事先调用 mmap 将服务端的用户空间的一段虚拟内存和内核空间的一段虚拟内存，映射到同一块物理内存上。这段物理内存，将作为一个缓冲区，接收来自不同客户端的数据。比如，客户端发送事务数据时，内核会通过 copy_from_user 将数据放进缓冲区，然后通知服务端。服务端只需要通过用户空间相应的虚拟地址，就能直接访问数据：
![[Binder的一次拷贝.webp]]
当 Android 的进程第一次通过 Binder 框架进行 IPC 时，就会触发 [ProcessState 的初始化](https://link.juejin.cn/?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-13.0.0_r1%3Aframeworks%2Fnative%2Flibs%2Fbinder%2FProcessState.cpp%3Bl%3D485%3Bbpv%3D1%3Bbpt%3D0 "https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/native/libs/binder/ProcessState.cpp;l=485;bpv=1;bpt=0")，调用 Binder 驱动的 mmap。
``` cpp
//frameworks/native/libs/binder/ProcessState.cpp
ProcessState::ProcessState(const char* driver)

      : mDriverName(String8(driver)),

        mDriverFD(-1),

        mVMStart(MAP_FAILED),

        mThreadCountLock(PTHREAD_MUTEX_INITIALIZER),

        mThreadCountDecrement(PTHREAD_COND_INITIALIZER),

        mExecutingThreadsCount(0),

        mWaitingForThreads(0),

        mMaxThreads(DEFAULT_MAX_BINDER_THREADS),

        mStarvationStartTimeMs(0),

        mForked(false),

        mThreadPoolStarted(false),

        mThreadPoolSeq(1),

        mCallRestriction(CallRestriction::NONE) {

    base::Result<int> opened = open_driver(driver);

  

    if (opened.ok()) {

        // mmap the binder, providing a chunk of virtual address space to receive transactions.

        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,

                        opened.value(), 0);

        if (mVMStart == MAP_FAILED) {

            close(opened.value());

            // *sigh*

            opened = base::Error()

                    << "Using " << driver << " failed: unable to mmap transaction memory.";

            mDriverName.clear();

        }

    }

  

#ifdef __ANDROID__

    LOG_ALWAYS_FATAL_IF(!opened.ok(), "Binder driver '%s' could not be opened. Terminating: %s",

                        driver, opened.error().message().c_str());

#endif

  

    if (opened.ok()) {

        mDriverFD = opened.value();

    }

}

static base::Result<int> open_driver(const char* driver) {

    int fd = open(driver, O_RDWR | O_CLOEXEC);

    if (fd < 0) {

        return base::ErrnoError() << "Opening '" << driver << "' failed";

    }

    int vers = 0;

    status_t result = ioctl(fd, BINDER_VERSION, &vers);

    if (result == -1) {

        close(fd);

        return base::ErrnoError() << "Binder ioctl to obtain version failed";

    }

    if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {

        close(fd);

        return base::Error() << "Binder driver protocol(" << vers

                             << ") does not match user space protocol("

                             << BINDER_CURRENT_PROTOCOL_VERSION

                             << ")! ioctl() return value: " << result;

    }

    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;

    result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);

    if (result == -1) {

        ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));

    }

    uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION;

    result = ioctl(fd, BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);

    if (result == -1) {

        ALOGE_IF(ProcessState::isDriverFeatureEnabled(

                     ProcessState::DriverFeature::ONEWAY_SPAM_DETECTION),

                 "Binder ioctl to enable oneway spam detection failed: %s", strerror(errno));

    }

    return fd;

}
```
## 参考链接
[mmap实现原理解析](https://blog.csdn.net/weixin_42937012/article/details/121269058)
[图解 Binder：内存管理 - 掘金](https://juejin.cn/post/7244734179828203579)
[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)j]
# Binder 驱动
```

```
## 参考链接
[Android源码分析 - Binder概述 - 掘金](https://juejin.cn/post/7059601252367204365)