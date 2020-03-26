### 情景

1. 使用主线程单例Handler的`post()`，想要实现全局都可以往主线程post代码。
2. 在`post()`中传入了一个`Runnable`对象，该`Runnable`对象在Activity中非静态，是局部匿名内部类的形式存在。

### 代码

``` kotlin
//单例类，提供一个主线程单例Handler。
object ExecutorsHelper {
    val mainThread = MainThreadExecutor()
    class MainThreadExecutor {

        private val handler = Handler(Looper.getMainLooper())

        fun execute(lifecycleOwner: LifecycleOwner, command: Runnable?) {
			//对生命周期组件进行处理，在其销毁时，手动将该Runnable从主线程消息队列中移除
            lifecycleOwner.lifecycle.addObserver(object : LifecycleEventObserver {
                override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event){
                    if (event == Lifecycle.Event.ON_DESTROY) {
                        lifecycleOwner.lifecycle.removeObserver(this)
                        handler.removeCallbacks(command)
                        LogUtils.d("Main2Activity","尝试将Runnable从全局handler中移除！")
                    }
                }
            })
			//post到主线程消息队列
            handler.post(command)
        }
    }
}
//发生内存泄露的Activity
class Main2Activity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main2)

        ExecutorsHelper.mainThread.execute(this, Runnable {
            thread {
                val sleepTime = SECOND * 5
                Thread.sleep(sleepTime)
                LogUtils.d("${this}中的线程执行完毕")
            }
            LogUtils.d("$this 中引用的runnable在消息队列中执行完毕")
        })
    }

    companion object {
        //1秒
        const val SECOND = 1000L
    }
}
//主Activity
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        btn.setOnClickListener {
            startActivity(Intent(this, Main2Activity::class.java))
        }
    }
}
```

当我从MainActivity不断地启动，销毁，启动，销毁Main2Activity时，发生了内存泄露。

原因：虽然我在Activity发生`onDestory（）`时将Runnable对象从主线程消息队列中移除（虽然移除的时候，Runnable早就执行完并且不在消息队列上了），但是由于Runnable代码块中启动了一个异步任务，这个异步任务等待了5秒，并且持有了外部对象`Main2Activity`的强引用。

---

补充：为什么`Main2Activity`中创建的`Thread`对象不会被回收，他不是没有被任何对象引用吗？

答：存活的线程对象本身就是一个GC root，他不会被回收，而JVM顺着该GC root找到了`Thread`强引用的`Main2Activity`对象，因此`Main2Activity`不会被回收。

*（我在这里也一直都不知道原来 a live Thread is a GC root这个概念的，当时学《深入理解java虚拟机》的时候是没有提到过这个概念的，这次也是刷新了认知了）*

![](https://upload-images.jianshu.io/upload_images/7177220-40cad7fa1c198e65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

https://www.yourkit.com/docs/java/help/gc_roots.jsp

---



此处解决方案：让Runnable作为静态内部类，如果需要一个外部类`Activity`的引用，用`WeakReference`和空引用判断。

改造后的`Main2Activity`

``` kotlin
class Main2Activity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main2)

        ExecutorsHelper.mainThread.execute(this, ARunnable(this))
    }

    class ARunnable(activity: Main2Activity) : Runnable {

        private val weakReference: WeakReference<Main2Activity> = WeakReference(activity)

        override fun run() {
            thread {
                val sleepTime = SECOND * 5
                Thread.sleep(sleepTime)
                LogUtils.d("${weakReference.get()?.toString()}中的线程执行完毕")
            }
            LogUtils.d("$this 中引用的runnable在消息队列中执行完毕")
        }
    }

    companion object {
        //1秒
        const val SECOND = 1000L
    }
}
```

不再发生内存泄漏。

---
**启示**:防止Handler内存泄漏，需要：
1. 将Handler做成静态内部类
2. 在控制器的`onDestroy()`中移除消息，例如`handler.removeCallbacks(...)`
3. `handler.post(Runnable)`中，但凡开启了异步任务的，都要将`Runnable`也做成静态内部类，且用`WeakReference`来获得控制器的引用。
