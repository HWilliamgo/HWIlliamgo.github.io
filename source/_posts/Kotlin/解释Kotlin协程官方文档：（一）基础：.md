本系列文章用于解释kotlin官方文档中的示例代码。希望能帮助到你。

### 基础

官方文档地址 https://www.kotlincn.net/docs/reference/coroutines/basics.html 



##### 作用域构建器

``` kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建一个协程作用域
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌 launch 之前输出
    }
    
    println("Coroutine scope is over") // 这一行在内嵌 launch 执行完毕后才输出
}
```



程序的运行结果：

```
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
```



解释：

``` kotlin
import kotlinx.coroutines.*

//main方法一启动就创建一个协程，因为这个协程直接把主线程阻塞了，就叫CoroutineBlock吧
fun main() = runBlocking { // this: CoroutineScope
    //创建一个新的协程，叫做CoroutinebA吧 1
    launch { 
        //挂起当前协程200ms 2
        delay(200L)
        println("Task from runBlocking")
    }
    
    //挂起当前协程 3
    coroutineScope { // 创建一个协程作用域
        //创建新一个新的协程,叫做CoroutineB吧 4
        launch {
            //挂起当前协程500ms 5
            delay(500L) 
            println("Task from nested launch")
        }
    
        //挂起当前协程100ms 6
        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌 launch 之前输出
    }
    
    println("Coroutine scope is over") // 这一行在内嵌 launch 执行完毕后才输出
}
```

我在上述代码中用数字做了标记，现在结合这些标记，逐行代码进行分析。

1. 1处，创建一个新的协程，叫他CoroutineA，并执行协程，立刻返回。

2. 然后执行到3，`coroutineScpoe()`是一个Kotlin协程库中的挂起函数，此时挂起当前协程，当前协程是什么？当前协程正在执行`runBlocking{}`函数中的代码块，因此此时外部的CoroutineBlock协程被挂起，并调用3处创建的代码块。

3. 此时走到4处，又创建了一个新的协程CoroutineB，因为不是挂起函数，所以创建协程后启动并从代码块返回。

4. 此时走到6处，遇到挂起函数`delay()`，那么挂起当前的协程CoroutineBlock。

5. 前面两个`launch()`方法启动了两个协程，分别是CoroutineA和CoroutineB，他们分别走到了 2处 和 5处。并各自调用了一个挂起函数`delay()`。

6. 仔细数一数，此时挂起的协程已经有三个了：分别是`JobBlock`，`JobA`和`JobB`。但是根据他们挂起函数挂起的时间，那么从挂起函数恢复的先后顺序是6处，2处，5处。

7. 于是根据挂起函数恢复的先后顺序，打印：

   ```
   Task from coroutine scope
   Task from runBlocking
   Task from nested launch
   ```

8. 还是回到3个协程挂起恢复的地方，先是6处的协程CoroutineBlcok恢复，那么他打印完后，会再等待当前协程作用于创建的协程CoroutineB完成后，再从`coroutineScope()`方法返回。而当协程CoroutineB执行完后，已经打印完了3行内容。此时从3处的挂起处恢复。

9. 继续执行协程CoroutineBlock，打印：

   ```
   Coroutine scope is over
   ```

   





这里我的疑问是：函数`coroutineScope()`创建了一个协程作用域并挂起，并等内部所有的子协程也完成了才立刻返回。

为什么要这么设计呢？是否有其他创建协程作用于的函数也是这样的功能呢？
