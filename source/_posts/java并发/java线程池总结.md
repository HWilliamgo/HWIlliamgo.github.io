>我学习线程池的第一篇讲的最好、最明白的文章：
[java&android线程池-Executor框架之ThreadPoolExcutor&ScheduledThreadPoolExecutor浅析（多线程编程之三）](http://blog.csdn.net/javazejian/article/details/50890554)


>另外一个是能快速帮你回忆的总结性文章，说的也不错：
[通俗易懂，各常用线程池执行的-流程图](https://juejin.im/post/5a28b37c6fb9a044fc44a103)

ThreadPoolExecutor的构造方法的参数：
```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        ......
    }
```
从中总结了一个非常非常重要的流程图：（没有理解这个，谈线程池怎么都谈不明白）
![](http://upload-images.jianshu.io/upload_images/7177220-d8241458820e32d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


