

---
> 以下内容完全来自于《java核心技术卷Ⅰ 》

当线程的run方法执行方法体中最后一条语句后,并经由执行return语句返回时,或者出现了在方法中没有捕获的异常时,线程将终止。在Java的早期版本中,还有一个stop方法,其他线程可以调用它终止线程。但是,这个方法现在已经被弃用了。

有一种可以强制线程终止的方法。然而, interrupt方法可以用来请求终止线程。

当对一个线程调用interrupt方法时线程的中断状态将被置位。这是每一个线程都具有的boolean标志。每个线程都应该不时地检查这个标志,以判断线程是否被中断。

要想弄清中断状态是否被置位，首先调用静态的Thread.currentThread方法获得当前线程，然后调用isInterrupted()方法：
```
while(!Thread.currentThread().isInterrupted() && more work to do){
  do more work
}
```
但是，如果线程被阻塞，就无法检测中断状态。这是产生InterruptedException异常的地方。当在一个被阻塞的线程（调用sleep或wait）上调用interrupt方法时，阻塞调用将会被InterruptedException异常终点。

没有任何语言方面的需求要求一个被中断的线程应该终止。中断一个线程不过是引起它的注意。被中断的线程可以决定如何响应中断。某些线程是如此重要以至于应该处理完异常后,继续执行,而不理会中断。但是,更普遍的情况是,线程将简单地将中断作为一个终止的请求。这种线程的run方法具有如下形式:
```
public void run(){
  try{
    ...
      while(!Thread.currentThread().isInterrupted() && more work to do){
          do more work
        }
  }catch(InterruptedException e){
    //thread was interrupted during sleep or wait
  }finally{
    cleanup , if required
  }
   //exiting the run method terminates the thread
}
```
如果在每次工作迭代之后都调用sleep方法(或者其他的可中断方法), isInterrupted检测既没有必要也没有用处。如果在中断状态被置位时调用sleep方法,它不会休眠。相反,它将清除这一状态(!)并抛出InterruptedException。因此,如果你的循环调用sleep,不会检测中断状态。相反,要如下所示捕获InterruptedException异常:
```
public void run(){
  try{
    ...
    while(more work to do）{
      do more work
      Thread.sleep(delay);
    }
  }catch(InterruptedException e){
     //thread was interrupted during sleep
  }finally{
    cleanup, if required
  }
  //exiting the run method terminateds the thread
}
```
**注释**:有两个非常类似的方法, interrupted和isInterrupted, interrupted方法是一个静态方法,它检测当前的线程是否被中断。而且,调用interrupted方法会清除该线程的中断 "状态。另一方面, isInterrupted方法是一个实例方法,可用来检验是否有线程被中断。调用这个方法不会改变中断状态。

在很多发布的代码中会发现InterruptedException异常被抑制在很低的层次上，像这样：
```
void mySubTask{
  ...
  try{sleep(delay)}
  catch(InterruptedExcpetion e{}//DON'T IGNORE!
  ...
}
```
不要这样做！如果不认为在catch子句中做这一处理有什么好处的话，仍然有两种合理的选择：
1. 在catch子句中调用Thread.currentThread().interrupt来设置中断状态。于是，调用者可以对其进行检测。
```
void mySubTask{
  ...
  try{sleep(delay)}
  catch(InterruptedExcpetion e{
    Thread.currentThread.interrupt();
  }
  ...
}
```
2. 或者，更好的选择是，用throws InterruptedException将异常抛出，留给调用者。  

void mySubTask{
  ...
  try{sleep(delay)}
  catch(InterruptedExcpetion e{}//DON'T IGNORE!
  ...
}



* `void interrupt()`
向线程发送中断请求,线程的中断状态将被设置为true,如果目前该线程被一个sleep调用阻塞,那么, InterruptedException异常被抛出。
* `static boolean interrupted()`
测试当前线程(即正在执行这一命令的线程)是否被中断。注意,这是一个静态方法。"这一调用会产生副作用-它将当前线程的中断状态重置为false。
* `boolean isInterrupted()`
测试线程是否被终止。不像静态的中断方法,这一调用不改变线程的中断状态。
* `static Thread currentThread()`
返回代表当前执行线程的Thread对象
