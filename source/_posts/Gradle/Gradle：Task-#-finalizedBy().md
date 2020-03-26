# Gradle：Task # finalizedBy()

> 这篇文章对Gradle的Task对象的finalizedBy()方法做一个介绍

### 1 官方文档

接口说明：

![](https://upload-images.jianshu.io/upload_images/7177220-c14619bc3420d618.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用示例：

https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:finalizer_tasks



官方对这个方法只有只言片语的介绍，以下是我自己的理解。

### 2 概述

`Task # finalizedBy(Object ... paths)`

为当前的task添加一个或若干个当前task结束后立马执行的task，且添加的task的执行顺序为无序。

### 3 实例

``` gro
task A {
    finalizedBy "B"
    finalizedBy "C"
    finalizedBy "D"
    finalizedBy "E"
    doLast {
        println "A"
    }
}
task B {
    doLast {
        println "B"
    }
}
task C {
    doLast {
        println "C"
    }
}
task D {
    doLast {
        println "D"
    }
}
task E {
    doLast {
        println "E"
    }
}
```

执行命令：gradlew A

打印：

> Task :A
> A

> Task :E
> E

> Task :D
> D

> Task :C
> C

> Task :B
> B

打印出来的结果的确表明，finalizedBy()添加的后续执行的task的执行顺序是不确定的。

无序的原因：

这里要看一下gradle的源码，`Task`的实现类是`AbstractTask`

，`finalizedBy()`源码为：

``` java
@Override
public Task finalizedBy(final Object... paths) {
    taskMutator.mutate("Task.finalizedBy(Object...)", new Runnable() {
        public void run() {
            finalizedBy.add(paths);
        }
    });
    return this;
}
```

`finalizedBy`变量为`DefaultTaskDependency finalizedBy`。

查看他的add方法：

``` java
//DefaultTaskDependency 
public DefaultTaskDependency add(Object... values) {
    for (Object value : values) {
        addValue(value);
    }
    return this;
}
```

``` java
private void addValue(Object dependency) {
    if (dependency == null) {
        throw new InvalidUserDataException("A dependency must not be empty");
    }
    getMutableValues().add(dependency);
}
```

``` java
public Set<Object> getMutableValues() {
    if (mutableValues == null) {
        mutableValues = new TaskDependencySet();
    }
    return mutableValues;
}
```

即`finalizedBy`是往一个Set类型的容器里面去添加task的，而Set在取的时候是无序的，所以后面执行就是无序的。

### 4 结论

1. finalizedBy(Object ... paths)方法的参数显示的是Object类型，但实际上用的时候要传递String类型字符串。
2. finalizedBy()添加的task在执行的时候是无序的。
3. finalizedBy()方法不能在Task执行的时候调用，即不能在doLast{}和doFirst{}里面调用，只能在Task构造的时候调用。
