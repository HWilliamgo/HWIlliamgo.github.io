## 接口Executor:
一个可以执行被提交的Runnable的对象。该接口提供了一种任务提交的解耦方案，体现在每个task如何运行，使用，调度等。
Executor接口并不严格要求task的执行过程是异步的，在最简单的情况下，一个Executor可以直接将task放在调用者的线程中执行：
```
class DirectExecutor implements Executor{
  public void execute(Runnable r){
    r.run();
  }
}
```
但更常见的情况一般是：task会被放置在其他线程而非调用者线程执行。
许多Executor接口的实现会给task如何以及何时被调用加上限制。
```
void execute(Runnable command);
```
Runnable可在新线程，线程池，或者调用者线程执行。

## 接口ExecutorService（Executor的拓展）
是一个提供了`termination`和返回Future来追踪一个或多个异步tasks的对象。

`termination`：指executor没有正在执行的task，没有等待执行的task，且没有新的task可以被接收。

`shutdown();`新的task不再被接收，已提交的task会执行完毕，重复调用该方法没有其他效果。该方法不会阻塞调用者线程来等待task去执行完毕，若要阻塞当前线程，该调用`awaitTermination();`

`shutdownNow();`尝试停止所有正在执行的task，停止正在等待的task，返回等待执行的task的List<Runnable>。但并不保证能成功停止正在执行的线程，因为该方法的实现一般就是用`Thread#interrupt()`，那么就要求线程本身就能正确根据`interrupt()`方法来停止自身。

`isShutdown()`返回true如果调用过`shutdown()`或者`shutdownNow()`。

`isTerminated();`如果全部task都在调用shutdown()后完成了，将返回true，若调用`isTerminated()`之前没有调用过`shutdown()`或`shutdownNow()`，该方法永远返回false。

`awaitTermination(long timeout,Timeunit unit`) throws InterruptedExeption;  阻塞调用者的线程，直到所有task在shutdown后完成，或者timeout超时，或者当前线程被interrupted。

`Future<T>  submit(Callable<T>)`提交一个有返回值的任务（Callble）并返回一个Future对象。可通过Future#get()来返回任务处理完成的结果。若要为获取一个任务的执行结果而阻塞当前调用者线程，可以
```
result=exec.submit(aCallable).get();
```
