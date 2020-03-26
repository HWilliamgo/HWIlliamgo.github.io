
![](https://upload-images.jianshu.io/upload_images/7177220-f0bd9b56a9d192a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ThreadPoolExecutor`对象可以通过`Executor`的工厂方法(newXXX)或者他自己的构造函数来创建。他是Java线程池的实现类。



使用：

``` kotlin
//Runnable
fun main(args: Array<String>) {
    val pool= Executors.newFixedThreadPool(3)
    pool.execute{
        Thread.sleep(2000)
        println("无返回值")
        pool.shutdown()
    }
}

//FutureTask
fun main(args: Array<String>) {
    val pool= Executors.newFixedThreadPool(3)
    val futureTask=FutureTask<String>(Callable<String>{
        Thread.sleep(2000)
        return@Callable "有返回值"
    })
    pool.submit(futureTask)
    //block current thread
    val result=futureTask.get()
    println(result)
	pool.shutdown()
}
```

一般用的比较多的都是第一种，创建`Runnable`实例，通过调用`execute()`或者`submit()`，交给线程池内部去处理。

第二种方法用的不多，下文浅析他的工作机制。

### submit()

`ThreadPoolExecutor`的`pool.submit(xxx)`方法来自于他的父类`AbstractExecutorService`。

``` java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

在这里可以看到，将传入的无论是`Runnable`还是`Callable`类型的变量，统一用一个`newTaskFor()`方法封装成了`RunnableFuture`对象，并且统一交给了`execute(Runnable command)`方法，这样就又回到了最熟悉的使用线程池的方式了。在`execute()`方法中，让线程池不同的实例去用不同的方法调度工作线程和任务队列。

在这里将`RunnableFuture`作为`Runnable`的子类型传递给`execute`，又作为`Future`的子类型从`submit()`方法返回。

他的实例化由`newTaskFor()`方法负责。

``` java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

全部返回了具体类型`FutureTask`的子类 。

### FutureTask

uml类图如下：

![](https://upload-images.jianshu.io/upload_images/7177220-b484f4af170a1a56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这里关注`FutureTask`的三个方法：

1. 构造方法
2. run()方法。（作为Runnable子类传递给线程池的直接执行方法）
3. get()方法。（客户端调用，阻塞当前线程，获取执行结果）



#### 1. 构造方法

``` java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
public FutureTask(Runnable runnable, V result) {
	//用Executors.callable将runnable转换成callable对象。
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

构造方法：注入Callable对象，将当前状态标志为NEW。

#### 2. get()方法

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        //阻塞当前线程
        s = awaitDone(false, 0L);
    //当线程从阻塞状态被唤醒，将结果返回。
    return report(s);
}

//Awaits completion or aborts on interrupt or timeout.
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
			//阻塞当前线程
            LockSupport.park(this);
    }
}
```

这里是`get()`-->`awaitDone()`-->`LockSupport.park(this)`。

在`LockSupport.park()`方法内部，获取当前线程，并阻塞。

那么当线程从阻塞状态唤醒时（run方法执行完毕主动唤醒，或者遇到异常被唤醒），调用`report(int s)`方法将执行结果返回。

``` java
/** The result to return or exception to throw from get() */
private Object outcome; // non-volatile, protected by state reads/writes

private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

读取实例变量`outcome`的值，并返回。

到这里就应该猜到，`outcome`的值的写入应该发生在`run()`方法里面了。

#### 3. run()方法

``` java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                //调用callback的方法，并获取执行结果
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);//将执行结果设置到实例变量中。
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

在这里直接调用`callback.call`方法执行其内部代码，然后将结果用`set()`保存起来

``` java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        //保存执行结果
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        //唤醒当前线程。
        finishCompletion();
    }
}

/**
 * Removes and signals all waiting threads, invokes done(), and
 * nulls out callable.
 */
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    //唤醒线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    done();
    callable = null;        // to reduce footprint
}
```





---

机制分析结束，在使用上面，`FutureTask`和`Runnable的`最大不同就是，可以用`get()`方法阻塞当前线程并获取执行结果。
