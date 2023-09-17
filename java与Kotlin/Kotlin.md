# 泛型
## 基本介绍
1. Java 允许实例化没有具体类型参数的泛型类。但 Kotlin 要求在使用泛型时需要**显式声明泛型类型**或者是**编译器能够类型推导出具体类型**，任何不具备具体泛型类型的泛型类都无法被实例化。
2.  类型通配符：`Java`中是`？`,` kotlin`是`*`；
3. 上届约束进一步细化其支持的类型：`T extends Object `，`kotlin`使用`:`代替`extends`没有指定上界约束的类型形参会默认使用 `Any?` 作为上界;
4.  不变：`List<Object>` 不是`List<String> `的父类型；
5. 协变/上界通配符: `List<String>`就是的`List<? extend Object>`子类型, `kotlin`中使用`out`表示;
6. 逆变/下界通配符：`List<? super String>`就是的`List<Object>`子类型, `kotlin`中使用`in`表示;
``` kotlin
  var stringList =  mutableListOf<String>()  
  var objectList:MutableList<Any> = mutableListOf()  
//    objectList = list // 编译报错  
  var anyString :MutableList<out Any> = stringList // 协变  
  var inString :MutableList<in String> = objectList //逆变
```
7. PESC：意为「Producer Extend Consumer Super」，当然在Kotlin中，这句话要改为「Consumer in, Producer out」。这个原则是从集合的角度出发的，其目的是为了实现集合的多态。
	-   如果只是从集合中读取数据，那么它就是个生产者，可以使用`extend`
    
	-   如果只是往集合中增加数据，那么它就是个消费者，可以使用`super`
    
	-   如果往集合中既存又取，那么你不应该用`extend`或者`super`
8. 定义型变的位置，分为使用处型变（对象定义）和声明处型变（类定义），`Java`不支持后者。
```kotlin
interface Source<in T, out R,out U> {  
    fun next(): R  //消费者 只能作为返回值
	fun execute(item: T)  // 生产者：方法参数
    fun getType(type: @UnsafeVariance U)  // 注解
}
```
9. 泛型擦除：todo
10. 获取泛型类型：
	-  反射//todo
	-  `inline`内联函数 + `reified`关键字（类型不擦除 )
```
inline fun <reified T : Activity> Activity.startActivity(context: Context) {  
startActivity(Intent(context, T::class.java))  
}  
  
// 调用  
startActivity<MainActivity>(context)
```

### 参考链接
- [ 一文读懂 Java 和 Kotlin 的泛型难点](https://juejin.cn/post/6935322686943920159#heading-2)
- [kotlin修炼指南7之泛型](https://mp.weixin.qq.com/s/9KjbLAB_99jvh80JGNKgYw)
- [换个姿势，十分钟拿下Java/Kotlin泛型](https://mp.weixin.qq.com/s/vSwx7fgROJcrQwEOW7Ws8A)
- [Kotlin泛型的型变之路](https://mp.weixin.qq.com/s/UkgUfdcKCEP8My7jAxOknQ)
# 协程
## 挂起函数
### 普通函数
``` java
suspend fun suspendFun():String{  
    delay(1000)  
    return "Joe"  
}
public static final Object suspendFun(@NotNull Continuation var0) {  
   Object $continuation;  
   label20: {  
      if (var0 instanceof <undefinedtype>) {  
         $continuation = (<undefinedtype>)var0;  
         if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {  
            ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;  
            break label20;  
         }  
      }  
  
      $continuation = new ContinuationImpl(var0) {  
         // $FF: synthetic field  
         Object result;  
         int label;  
  
         @Nullable  
         public final Object invokeSuspend(@NotNull Object $result) {  
            this.result = $result;  
            this.label |= Integer.MIN_VALUE;  
            return CoroutinesDemo1Kt.suspendFun(this);  
         }  
      };  
   }  
  
   Object $result = ((<undefinedtype>)$continuation).result;  
   Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();  
   switch (((<undefinedtype>)$continuation).label) {  
      case 0:  
         ResultKt.throwOnFailure($result);  
         ((<undefinedtype>)$continuation).label = 1;  
         if (DelayKt.delay(1000L, (Continuation)$continuation) == var3) {  
            return var3;  
         }  
         break;  
      case 1:  
         ResultKt.throwOnFailure($result);  
         break;      default:  
         throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");  
   }  
  
   return "Joe";  
}
```
一个普通的挂起函数，编译后会在原来的参数之外额外添加一个Continuation的参数，
### 高阶函数
``` kotlin
val suspendFunc: suspend String.() -> Float = {  
    delay(1000)  
    1.0f  
}
private static final Function2 suspendFunc = (Function2)(new Function2((Continuation)null) {  
   int label;  
  
   @Nullable  
   public final Object invokeSuspend(@NotNull Object $result) {  
      Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();  
      switch (this.label) {  
         case 0:  
            ResultKt.throwOnFailure($result);  
            this.label = 1;  
            if (DelayKt.delay(1000L, this) == var2) {  
               return var2;  
            }  
            break;  
         case 1:  
            ResultKt.throwOnFailure($result);  
            break;         default:  
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");  
      }  
  
      return Boxing.boxFloat(1.0F);  
   }  
  
   @NotNull  
   public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {  
      Intrinsics.checkNotNullParameter(completion, "completion");  
      Function2 var3 = new <anonymous constructor>(completion);  
      return var3;  
   }  
  
   public final Object invoke(Object var1, Object var2) {  
      return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);  
   }  
});
```
对于高阶函数，会生成一个类继承`SuspendLambda`，实现对应的`Function`接口，示例中为`Function1`。
普通函数与高阶函数的都是BaseContinuationImpl，
``` kotlin
public final override fun resumeWith(result: Result<Any?>) {  
    // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume  
    var current = this  
    var param = result  
    while (true) {  
        // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure  
        // can precisely track what part of suspended callstack was already resumed        probeCoroutineResumed(current)  
        with(current) {  
            val completion = completion!! // fail fast when trying to resume continuation without completion  
            val outcome: Result<Any?> =  
                try {  
                    val outcome = invokeSuspend(param)  
                    if (outcome === COROUTINE_SUSPENDED) return  
                    Result.success(outcome)  
                } catch (exception: Throwable) {  
                    Result.failure(exception)  
                }  
            releaseIntercepted() // this state machine instance is terminating  
            if (completion is BaseContinuationImpl) {  
                // unrolling recursion via loop  
                current = completion  
                param = outcome  
            } else {  
                // top-level completion reached -- invoke and return  
                completion.resumeWith(outcome)  
                return  
            }  
        }  
    }  
}

```
## CoroutineScope
结构化并发
结构化并发 来解决协程不可控的问题：
1. 可以取消协程任务；
2. 协程任务正在执行中，可以追踪任务的状态；
3. 协程任务正在执行中，如果出现异常，可以发出信息；
4. 管理协程的生命的周期；
#### 常用的子类
GlobalScope: 进程级别，跟随App进程；
MainScope: 在Activity中使用，可以在onDestroy中使用
ViewModelScope:绑定ViewModel生命周期；
LifecycleScope: 跟随Lifecycle生命周期，绑定Activity/Fragment的生命周期
Scope如何实现生命周期管理？
## CoroutineContext
### 简介
保存协程上下文的自定义集合，主要由以下4个`Element`组成：
- `Job`：协程的唯一标识，用来控制协程的生命周期(`new、active、completing、completed、cancelling、cancelled`)；
- `CoroutineDispatcher`：协程调度器，指定协程运行的线程(`IO、Default、Main、Unconfined`);
- `CoroutineName`: 指定协程的名称，默认为coroutine;
- `CoroutineExceptionHandler`: 指定协程的异常处理器，用来处理未捕获的异常.
#### 数据结构
##### Element
``` kotlin
public interface Key<E : Element>

public interface Element : CoroutineContext {  
    /**  
     * A key of this coroutine context element.     */    public val key: Key<*>  
  
    public override operator fun <E : Element> get(key: Key<E>): E? =  
        @Suppress("UNCHECKED_CAST")  
        if (this.key == key) this as E else null  
  
    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =  
        operation(initial, this)  
  
    public override fun minusKey(key: Key<*>): CoroutineContext =  
        if (this.key == key) EmptyCoroutineContext else this  
}
```
CoroutineContext中的元素都必须实现Element接口，每个元素都有唯一的Key, 原来检索元素。
##### `plus`
``` kotlin
public operator fun plus(context: CoroutineContext): CoroutineContext =  
    if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation  
        context.fold(this) { acc, element ->  
            val removed = acc.minusKey(element.key)  
            if (removed === EmptyCoroutineContext) element else {  
                // make sure interceptor is always last in the context (and thus is fast to get when present)  
                val interceptor = removed[ContinuationInterceptor]  
                if (interceptor == null) CombinedContext(removed, element) else {  
                    val left = removed.minusKey(ContinuationInterceptor)  
                    if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else  
                        CombinedContext(CombinedContext(left, element), interceptor)  
                }  
            }  
        }
```
1. `plus EmptyCoroutineContext` ：`Dispatchers.Main + EmptyCoroutineContext` 结果:`Dispatchers.Main`。
2. `plus` 相同类型的`Element`：`CoroutineName("c1") + CoroutineName("c2")`结果: `CoroutineName("c2")`。相同类型的直接替换掉。
3. `plus`方法的调用方没有`Dispatcher`相关的Element：`CoroutineName("c1") + Job()`结果:`CoroutineName("c1") <- Job`。头插法被plus的(`Job`)放在链表头部
4. `plus`方法的调用方只有`Dispatcher`相关的`Element` ：`Dispatchers.Main + Job()`结果:`Job <- Dispatchers.Main`。虽然是头插法，但是`ContinuationInterceptor`必须在链表头部。
5. `plus`方法的调用方是包含`Dispatcher`相关Element的链表： `Dispatchers.Main + Job() + CoroutineName("c5")`结果:`Job <- CoroutineName("c5") <- Dispatchers.Main`。Dispatchers.Main在链表头部，其它的采用头插法。
## CoroutineStart
## launch
### 相关对象的创建的过程
``` kotlin
public fun CoroutineScope.launch(  
    context: CoroutineContext = EmptyCoroutineContext,  
    start: CoroutineStart = CoroutineStart.DEFAULT,  
    block: suspend CoroutineScope.() -> Unit  
): Job {  
    val newContext = newCoroutineContext(context)  // 1
    val coroutine = if (start.isLazy)  
        LazyStandaloneCoroutine(newContext, block) else  
        StandaloneCoroutine(newContext, active = true)  
    coroutine.start(start, coroutine, block)  //2
    return coroutine  
}
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =  
    when (this) {  
        DEFAULT -> block.startCoroutineCancellable(receiver, completion)  
        ATOMIC -> block.startCoroutine(receiver, completion)  
        UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)  
        LAZY -> Unit // will start lazily  
    }
    
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(  
    receiver: R, completion: Continuation<T>,  
    onCancellation: ((cause: Throwable) -> Unit)? = null  
) =  
    runSafely(completion) {  
        createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit), onCancellation)  
    }

public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)//3
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}

// ContinuationImpl //4
public fun intercepted(): Continuation<Any?> =  
    intercepted  
        ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)  
            .also { intercepted = it }
//CoroutineDispatcher
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =  
    DispatchedContinuation(this, continuation)		
```
1. 与`CoroutineScope`组合生成新的`CoroutineContex`默认为 `Dispatchers.Default`；
2. 创建`StandaloneCoroutine`，其实现`Job`接口，通过它启动协程，并关联父`job`；
3. ` block `为高阶函数，继承自`SuspendLambda`，实现`Function2`接口，`create`方法，返回自身对象；
4. `intercepted`：`ContinuationImpl`是`SuspendLambda`  父类，持有的`CoroutineContext`为`StandaloneCoroutine+DefaultScheduler`，后者属于`ContinuationInterceptor`；
### 工作流程
``` kotlin
//DispatchedContinuation //1
inline fun resumeCancellableWith(  
    result: Result<T>,  
    noinline onCancellation: ((cause: Throwable) -> Unit)?  
) {  
    val state = result.toState(onCancellation)  
    if (dispatcher.isDispatchNeeded(context)) {  
        _state = state  
        resumeMode = MODE_CANCELLABLE  
        dispatcher.dispatch(context, this)  
    } else {  
        executeUnconfined(state, MODE_CANCELLABLE) {  
            if (!resumeCancelled(state)) {  
                resumeUndispatchedWith(result)  
            }  
        }  
    }  
}	
// CoroutineScheduler
fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {  
    trackTask()   
    val task = createTask(block, taskContext)  
    val isBlockingTask = task.isBlocking  
    val stateSnapshot = if (isBlockingTask) incrementBlockingTasks() else 0  
    val currentWorker = currentWorker()  
    val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)  
    if (notAdded != null) {  
        if (!addToGlobalQueue(notAdded)) {  
            throw RejectedExecutionException("$schedulerName was terminated")  
        }  
    }  
    val skipUnpark = tailDispatch && currentWorker != null  
    if (isBlockingTask) {  
        signalBlockingWork(stateSnapshot, skipUnpark = skipUnpark)  
    } else {  
        if (skipUnpark) return  
        signalCpuWork()  
    }  
}


//DispatchedTask
public final override fun run() {  
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching  
    val taskContext = this.taskContext  
    var fatalException: Throwable? = null  
    try {  
        val delegate = delegate as DispatchedContinuation<T>  
        val continuation = delegate.continuation  
        withContinuationContext(continuation, delegate.countOrElement) {  
            val context = continuation.context  
            val state = takeState() // NOTE: Must take state in any case, even if cancelled  
            val exception = getExceptionalResult(state)  
		      val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null  
            if (job != null && !job.isActive) {  
                val cause = job.getCancellationException()  
                cancelCompletedResult(state, cause)  
                continuation.resumeWithStackTrace(cause)  
            } else {  
                if (exception != null) {  
                    continuation.resumeWithException(exception)  
                } else {  
                    continuation.resume(getSuccessfulResult(state))  
                }  
            }  
        }  
    } catch (e: Throwable) {  
        // This instead of runCatching to have nicer stacktrace and debug experience  
        fatalException = e  
    } finally {  
        val result = runCatching { taskContext.afterTask() }  
        handleFatalException(fatalException, result.exceptionOrNull())  
    }  
}
```
1.  `resumeCancellableWith` : `dispatcher` 为线程池代理类，内部持有线程池，此时直接将任务交给线程池去分配；
2. `run`: `DispatchedContinuation`的continuation 对象，为`SuspendLambda` ，也就是我们`launch block`中的业务逻辑；

## delay 
``` kotlin
public suspend fun delay(timeMillis: Long) {  
    if (timeMillis <= 0) return // don't delay  
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->  
        // if timeMillis == Long.MAX_VALUE then just wait forever like awaitCancellation, don't schedule.  
        if (timeMillis < Long.MAX_VALUE) {  
            cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)  
        }  
    }  
}

public suspend inline fun <T> suspendCancellableCoroutine(  
    crossinline block: (CancellableContinuation<T>) -> Unit  
): T =  
    suspendCoroutineUninterceptedOrReturn { uCont ->  
        val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)  
        /*  
         * For non-atomic cancellation we setup parent-child relationship immediately         * in case when `block` blocks the current thread (e.g. Rx2 with trampoline scheduler), but         * properly supports cancellation.         */        cancellable.initCancellability()  
        block(cancellable)  
        cancellable.getResult()  
    }
```
### 协程的取消
1. yield  /isActive /ensureActive
2. 取消之后 资源无法释放：
	- try catch  cancel异常的处理；
	- use函数释放
3. 子协程取消（内部会抛出JobCancllationException），不会影响父协程的工作
### 协程的超时任务
### 协程的异常处理
异常会传递给父协程
### SuperVisorJob/supervisorJobScope
### 草稿
4. join与await 10秒与19秒？
5. supervisorScope 与coroutineScope
7. 大写的函数 ：简单工厂设计模式
8. async 立即开始调度 返回值和异常 等待await
9. 只有顶级协程才能处理异常？ExceptionHandler
10. 全局异常处理？自定义服务
11. flow与Rxjava
12. flow 冷流？
## flow
1. Flow上下文保存机制？，上下文保持一致？

# 语法糖
## inline noinline crossinline
### inline
inline是用来修饰函数的，在编译时，将该方法的函数类型的参数的方法体内联到方法调用处；
### noinline
noinline 用来修饰函数类型参数，内联后，该参数不在是函数类型，无法作为函数类型参数传递给其他函数；
### crossinline
**Kotlin中规定，在非内联函数中，lambda 表达式是不允许使用return返回。** crossline修饰参数，搭配inline使用
如果这个函数类型的参数直接在其他非内联中调用，从形式上来说变成了可以直接返回，编译报错；crossinline的作用仅仅是当有被这个修饰的参数会告诉IDE来检查你写的代码中有没有包含return，假如有的话会编译不过，就是这么简单暴力。
### 参考连接
[Kotlin的inline、noinline、crossinline全面分析2 - 掘金](https://juejin.cn/post/7050729336080433188)
## 委托属性

