# 计算机网络体系结构
## OSI模型
![[OSI模型.png]]
### TCP/IP模型
#### 三次握手
1. 第一次握手：客户端将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给服务器端，客户端进入SYN_SENT状态，等待服务器端确认。
2. 第二次握手：服务器端收到数据包后由标志位SYN=1知道客户端请求建立连接，服务器端将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给客户端以确认连接请求，服务器端进入SYN_RCVD状态。
3. 第三次握手：客户端收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给服务器端，服务器端检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，客户端和服务器端进入ESTABLISHED状态，完成三次握手，随后客户端与服务器端之间可以开始传输数据了。
#### 四次挥手
### 一次完整的http请求过程
## 草稿
1. 几种线程池的区别？
2. NIO的原理
4. select poll epoll的区别
5. 每入一次内核态 上下文切换2次
6. UDP 应用自己来实现可靠型传输 UDT 
7. Protobuf
## 最后一节课
1. 同步与异步，阻塞与非阻塞
2. 中断：暂停 先处理中断程序
# NIO
## Selector Channel Buffer
## epoll
epoll是Linux中最高效的IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作
# IO多路复用
## select/poll/epoll
### select
阻塞函数：将用户态的FD 拷贝到内核态
缺点：
1.  数量限制1024
2. fdset不可用
3. 拷贝时需要内核态到用户态的切换
4. 每次都需要重遍历
### poll
与select逻辑类似，但是没有数量限制
### epoll
# Okio
Okio的优势是设计了Segment作为数据的缓冲区，同时Segment是可以回收和复用的，减少内存消耗，提高了内存利用率。
## 以Okhttp的读为例
``` kotlin
//RealConnection
private fun connectSocket(  
    connectTimeout: Int,  
    readTimeout: Int,  
    call: Call,  
    eventListener: EventListener  
) {  
   ...  
   
    try {  
    // 1. 建立连接时，封装对应的Sink和Source对象
        source = rawSocket.source().buffer()  //RealBufferedSource
        sink = rawSocket.sink().buffer()   // RealBufferedSink
    } catch (npe: NullPointerException) {  
        if (npe.message == NPE_THROW_WITH_NULL) {  
            throw IOException(npe)  
        }  
    }  
}

//Http1ExchangeCodec
//2. 写入请求行和请求头信息
fun writeRequest(headers: Headers, requestLine: String) {  
  check(state == STATE_IDLE) { "state: $state" }  
  sink.writeUtf8(requestLine).writeUtf8("\r\n")  
  for (i in 0 until headers.size) {  
    sink.writeUtf8(headers.name(i))  
        .writeUtf8(": ")  
        .writeUtf8(headers.value(i))  
        .writeUtf8("\r\n")  
  }  
  sink.writeUtf8("\r\n")  
  state = STATE_OPEN_REQUEST_BODY  
}
//RealBufferedSink
internal inline fun RealBufferedSink.commonWriteUtf8(string: String): BufferedSink {  
  check(!closed) { "closed" }  
  buffer.writeUtf8(string)  
  return emitCompleteSegments()  
}
```
## Segment
``` kotlin 
internal class Segment {  
   val data: ByteArray  
   var pos: Int = 0 var limit: Int = 0  
  
    var shared: Boolean = false  
  
     var owner: Boolean = false  
  
    var next: Segment? = null  
  
	  var prev: Segment? = null  
  
  
    fun sharedCopy(): Segment {  
        shared = true  
        return Segment(data, pos, limit, true, false)  
    }  
  
    fun unsharedCopy() = Segment(data.copyOf(), pos, limit, false, true)  
  
    fun pop(): Segment? {  
        val result = if (next !== this) next else null  
        prev!!.next = next  
        next!!.prev = prev  
        next = null  
        prev = null  
        return result  
    }  
  
  
    fun push(segment: Segment): Segment {  
        segment.prev = this  
        segment.next = next  
        next!!.prev = segment  
        next = segment  
        return segment  
    }  
  
  
    fun split(byteCount: Int): Segment {  
        require(byteCount > 0 && byteCount <= limit - pos) { "byteCount out of range" }  
        val prefix: Segment  
  
        if (byteCount >= SHARE_MINIMUM) {  
            prefix = sharedCopy()  
        } else {  
            prefix = SegmentPool.take()  
            data.copyInto(prefix.data, startIndex = pos, endIndex = pos + byteCount)  
        }  
  
        prefix.limit = prefix.pos + byteCount  
        pos += byteCount  
        prev!!.push(prefix)  
        return prefix  
    }  
    fun compact() {  
        check(prev !== this) { "cannot compact" }  
        if (!prev!!.owner) return // Cannot compact: prev isn't writable.  
        val byteCount = limit - pos  
        val availableByteCount = SIZE - prev!!.limit + if (prev!!.shared) 0 else prev!!.pos  
        if (byteCount > availableByteCount) return // Cannot compact: not enough writable space.  
        writeTo(prev!!, byteCount)  
        pop()  
        SegmentPool.recycle(this)  
    }  
    fun writeTo(sink: Segment, byteCount: Int) {  
        check(sink.owner) { "only owner can write" }  
        if (sink.limit + byteCount > SIZE) {  
            // We can't fit byteCount bytes at the sink's current position. Shift sink first.  
            if (sink.shared) throw IllegalArgumentException()  
            if (sink.limit + byteCount - sink.pos > SIZE) throw IllegalArgumentException()  
            sink.data.copyInto(sink.data, startIndex = sink.pos, endIndex = sink.limit)  
            sink.limit -= sink.pos  
            sink.pos = 0  
        }  
  
        data.copyInto(  
            sink.data, destinationOffset = sink.limit, startIndex = pos,  
            endIndex = pos + byteCount  
        )  
        sink.limit += byteCount  
        pos += byteCount  
    }  
  
    companion object {  
        const val SIZE = 8192  
        const val SHARE_MINIMUM = 1024  
    }  
}

```

# OKHttp源码
## 分发器：Dispatcher
内部维护队列与线程池，完成请求调配
``` kotlin
class Dispatcher constructor() {  
    var maxRequests = 64  // 最大请求数
    var maxRequestsPerHost = 5  //同一主机的最大请求数
    private var executorServiceOrNull: ExecutorService? = null  // 执行异步请求的线程池
    private val readyAsyncCalls = ArrayDeque<RealCall.AsyncCall>()  //异步请求等待执行队列
    private val runningAsyncCalls = ArrayDeque<RealCall.AsyncCall>()  // 异步请求正在执行队列
    private val runningSyncCalls = ArrayDeque<RealCall>()  // 同步请求正在执行队列
}
```
### 异步请求
``` java
internal fun enqueue(call: AsyncCall) {  
  synchronized(this) {  
	// 1 立即加入等待执行队列
    readyAsyncCalls.add(call) // 1
    if (!call.call.forWebSocket) {  
      val existingCall = findExistingCallWithHost(call.host)  
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)  
    }  
  }  
  promoteAndExecute()  
}

private fun promoteAndExecute(): Boolean {  
  this.assertThreadDoesntHoldLock()  
  
  val executableCalls = mutableListOf<AsyncCall>()  
  val isRunning: Boolean  
  synchronized(this) {  
    val i = readyAsyncCalls.iterator() 
    // 遍历等待执行队列，找到所有可以执行的请求； 
    while (i.hasNext()) {  
      val asyncCall = i.next()  
	   //正在执行的请求数>64，直接结束循环
      if (runningAsyncCalls.size >= this.maxRequests) break
      //正在执行的请求中同一Host的请求超过5个，获取下一个请求
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue 
	  // 移除正在执行队列
      i.remove()  
      asyncCall.callsPerHost.incrementAndGet()  
      executableCalls.add(asyncCall)  
      // 加入到正在执行队列
      runningAsyncCalls.add(asyncCall)  
    }  
    isRunning = runningCallsCount() > 0  
  }  
  
  for (i in 0 until executableCalls.size) {  
    val asyncCall = executableCalls[i]  
    // 立即执行所有满足条件的请求
    asyncCall.executeOn(executorService)  
  }  
  
  return isRunning  
}
// 请求执行完成
internal fun finished(call: AsyncCall) {  
  call.callsPerHost.decrementAndGet()  
  finished(runningAsyncCalls, call)  
}
internal fun finished(call: RealCall) {  
  finished(runningSyncCalls, call)  
}  
  
private fun <T> finished(calls: Deque<T>, call: T) {  
  val idleCallback: Runnable?  
  synchronized(this) {  
  // 移除正在执行队列
    if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")  
    idleCallback = this.idleCallback  
  }  
  
  val isRunning = promoteAndExecute()  
  
  if (!isRunning && idleCallback != null) {  
    idleCallback.run()  
  }  
}
```

^41ea02

### 同步请求
``` kotlin 
@Synchronized internal fun executed(call: RealCall) {  
  runningSyncCalls.add(call)  
}
```
同步请求不需要线程池，也不存在任何限制，分发器只是做简单的记录；
## 拦截器
完成整个请求过程
