> 文章主题内容来自`Gradle官方文档`的`Understanding the Build Lifecycler`章节。通读完该章节，大大加深了我对task对象，project对象，gradle.build脚本和project对象的关系等这3个概念的理解。
>
> 官方文档地址：https://docs.gradle.org/current/userguide/build_lifecycle.html#sub:building_the_tree

![](https://upload-images.jianshu.io/upload_images/7177220-dddeef0781734dae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 构建的不同阶段

一个`gradle`的构建有3个不同的阶段

1. 初始化（Initialization）

   Gradle支持单和多project的构建。在初始化阶段，gradle决定了哪一个或哪些project将要参与到这次构建，并且为每个project创建一个`Project`对象。（注意，一个project对应一个build.gradle文件）

2. 安装（Configuration）

   在这个阶段，`Project`对象被安装(个人猜测是执行Project对象的构造函数)。所有参与到这次构建的build.gradle脚本文件都会被执行。

3. 执行（Execution）

   在此阶段，gradle将会决定在安装(Configuration)阶段所创建和装配的tasks的哪些子集tasks要被执行。被执行的那些task是通过`gradle`命令的参数中的task的名字和当前在哪个目录下来决定的。gradle然后执行每一个被选中的task。

## settings.gradle

1. 除了`build.gradle`脚本之外，gradle还定义了一个`settings.gradle`文件。这个文件会在初始化(initialization)阶段被执行。
2. 一个multi-project的构建必须有一个`settings.gradle`文件在根目录。因为后者定义了哪些project(也就是build.gradle脚本)会参与到这个构建。当然，单project(单个build.gradle脚本)的情况下，`settings.gradle`可有可无。

## 3个不同阶段在构建中的执行顺序
### 1. 定义了3个task来分别查看他们执行时打印的情况

1. `settings.gradle`文件

   ```groovy
   println('initialization：settings.gradle被执行')
   ```

2. `build.gradle`文件

   ```groovy
   println('configuration：build.gradle')
   task configured{
       println('configuration:task configured')
   }
   task A{
       println('configuration:task A')
       doLast{
           println '这里执行task:A#doLast'
       }
   }
   task B {
       doFirst{
           println '这里执行task:B#doFirst'
       }
       doLast{
           println '这里执行task:B#doLast'
       }
       println 'configuration:task B'
   }
   ```

3. `$ gradle configured`

   ``` shell
   william@localhost:~/IdeaProjects/1$ gradle configured
   initialization：settings.gradle被执行
   
   > Configure project : 
   configuration：build.gradle
   configuration:task configured
   configuration:task A
   configuration:task B
   ```

4. `$ gradle A`

   ``` shell
   initialization：settings.gradle被执行
   
   > Configure project : 
   configuration：build.gradle
   configuration:task configured
   configuration:task A
   configuration:task B
   
   > Task :A 
   这里执行task:A#doLast
   ```

5. `$ gradle B`

   ```shell
   并被添加到Project对象的一个字段TaskContainer tasks中，initialization：settings.gradle被执行
   
   > Configure project : 
   configuration：build.gradle
   configuration:task configured
   configuration:task A
   configuration:task B
   
   > Task :B 
   这里执行task:B#doFirst
   这里执行task:B#doLast
   ```

6. `$ gradle A B`

   ```shell
   initialization：settings.gradle被执行
   
   > Configure project : 
   configuration：build.gradle
   configuration:task configured
   configuration:task A
   configuration:task B
   
   > Task :A 
   这里执行task:A#doLast
   
   > Task :B 
   这里执行task:B#doFirst
   这里执行task:B#doLast
   ```

### 2. 结论

1. 初始化阶段执行settings.gradle脚本中的内容，

2. 安装阶段执行build.gradle脚本中除了`task.doLast(Closure c)`和`task.doFirst(Closure c)`中的所有内容。因为在安装阶段时，该`build.gradle`文件对应的`Project`对象已经创建，此时安装阶段对应的就是`Project`对象执行其构造方法。而上述代码中在`build.gradle`中定义了3个task，那么在安装阶段，这3个task就会被创建为3个task对象实例，并且被加入到`Project`对象的`TaskContainer`容器中，此后，这3个task实例就作为`Project`对象的属性，可以直接使用了。

3. 执行阶段，根据输入的task的名称和相关依赖（如果存在），去遍历执行对应的task对象中

   ```java
   private List<ContextAwareTaskAction> actions=new ArrayList<>();
   ```

   的所有的方法，而当我们调用`doLast`或者`doFirst`都是往那个`ArrayList`的头尾插入由我们传入的闭包而转化成的`Action`接口的对象。而`Action`接口长这样：

   ```java
   @HasImplicitReceiver
   public interface Action<T> {
       void execute(T t);
   }
   ```

   那么到执行阶段，这个task的`Action`的容器`actions`就会被遍历并调用每个`Action`的`execute(T t)`方法。



## 响应build.gradle脚本的生命周期

### 1. Project的安装

build.gradle中的内容依靠`Project`对象的构造方法来安装`Project`对象，其中有两个回调暴露给我们，分别是

```
afterEvalueate(Closure closure)
beforeEvalueate(Closure closure)
```

它们属于`Project`接口中就定义了的方法，因此直接用就行

build.gradle：

``` groovy
afterEvaluate {
    if (it.hasProperty('group')) {
        println('has group')
        it.task('B'){
            doLast{
                println 'execute B'
            }
        }
    } else {
        println('do not have group')
    }
}
task A {
    println(' configure A')
}
```

执行task B

```shell
william@localhost:~/IdeaProjects/1$ gradle B

> Configure project : 
configure A
has task A

> Task :B 
execute B


BUILD SUCCESSFUL in 0s
1 actionable task: 1 executed
```

执行结果如上，`Project#afterEvaluated(Closure closure)`方法会在`Project`对象全部安装完之后被调用，这也是其构造方法暴露给客户端的回调，那么`Project`对象的构造方法的伪代码如下：

```java
public ProjectImpl(){
    runClosureBeforeEvalueation();//执行安装前的闭包
    configureCodeInSript();//安装
    runClosureAfterEvalueation();//执行安装后的闭包    
}
```

### 2. Task的创建

在一个task对象被添加到一个`Project`对象后，可以立刻收到一个回调。

用`tasks.whenTaskAdded(Closure closure)`就可以办到

build.gradle：

```groovy
tasks.whenTaskAdded {
    println(it.name)
}
task A {
    println('configure A')
}
task B {
    println 'configure B'
}
```

执行task A

```shell
william@localhost:~/IdeaProjects/1$ gradle A

> Configure project : 
A
configure A
B
configure B
```

### 3.Task的执行

```groovy
gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
    }
    @Override
    void afterExecute(Task task, TaskState state) {
    }
})

gradle.taskGraph.beforeTask {
}
gradle.taskGraph.afterTask {
}
```

上面的接口的方法和下面的两个闭包方法的作用都是一样的，我认为他们相比于`Task#doFirst`或者`Task#doLast`方法的优点在于，`TaskExecutionGraph`的这几个方法，是为每一个安装了的task对象都插入一段回调，这段回调在每一个task执行前后被调用。

---

个人总结：

1. 在执行build.gradle之前，他所对应的`Project`对象已然创建，猜测是在执行`settings.gradle`时已经创建了。
2. build.gradle中的所有代码，都作为`Project`对象的构造函数一部分而插入构造函数。
3. 在build.gradle脚本中创建的所有Task对象都在创建后都被`Project`对象的`TaskContainer`这个容器对象引用。
4. 命令行每执行一次gradle命令，就创建一个程序，从类似java的`main`方法开始运行。创建`Project`对象，执行build.gradle脚本来安装(在他的构造方法中)，而gradle命令后面跟随的是task的名字，此时装配好`Project`对象后，根据传入的task的名字来去直接执行这个task。当task执行完毕输出结果后，程序结束，`main()`方法结束。
5. 当直接执行gradle命令而不加task，也会依然会执行build.gradle中为`Project`装配而存在的代码，即此时`Project`对象存在，但是只是不去execute task了。

**以上，为总结和对gradle运行原理的猜想，待考证更多资料后证实之。**
