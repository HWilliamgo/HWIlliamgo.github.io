
### 1 闭包概述

闭包，Clousure。在Java中表现为Lambda表达式，在Koltin中表现为函数类型变量（或者也叫做Lambda表达式）。

其定义：[java为什么匿名内部类的参数引用时final？ - 胖胖的回答](https://www.zhihu.com/question/21395848/answer/110829597)

1. 一个依赖于自由变量的函数 

2. 处在含有这些自由变量的一个外围环境

3. 这个函数能够访问外围环境里的自由变量



### 2 Kotlin和Java的闭包对比

在java8中，java中的闭包类型依然不能直接地修改外部环境的变量的引用，因此就有了众所周知的：匿名内部类的实例对外部环境的变量的访问，都需要让外部环境的变量加上`final`关键字。

在Kotin中，这一点被改进了，Kotlin中的闭包是功能健全的闭包。



### 3 代码

下面用简单的代码来对比java和kotlin的闭包类型变量在访问外部环境变量时的差异。

先定义了一个类型：鸟

``` kotlin
class Bird(private val wordToSing: String) {
    fun sing() {
        log("I sing $wordToSing")
    }
}
```

#### 3.1 Kotlin

``` kotlin
fun main() {
    var bird: Bird? = Bird("i 'm a testing bird.")

    val fun1 = {
        bird?.sing()
        bird = null
    }

    val fun2 = {
        if (bird != null) {
            log("bird is alive")
        } else {
            log("bird is destroyed")
        }
    }

    fun2()
    fun1()
    fun2()
}
```

在上述代码中，有两个闭包：`fun1`和`fun2`，他们的外部环境是`main()`函数，外部环境中的变量是`var bird`变量，那么两个闭包中都自由地对这个外部变量进行了读写。跑上述代码，打印：

``` 
bird is alive
I sing i 'm a testing bird.
bird is destroyed
```

#### 3.2 Java

``` java
public class Test {
    public static void main(String[] args) {
        Bird bird = new Bird("i 'm a testing bird");

        Runnable fun1 = () -> {
            bird.sing();
            bird = null;
        };

        Runnable fun2 = () -> {
            if (bird != null) {
                UtilsKt.log("bird is alive");
            } else {
                UtilsKt.log("bird is destroyed");
            }
        };

        fun2.run();
        fun1.run();
        fun2.run();
    }
}
```

用了和`Kotlin`完全一样的逻辑，编译期就报错了，IDEA提示我`variable used in a lambda expression should be final or effectively final`。

但是，给`bird`变量加上final之后，在`fun1`函数里面对`bird=null`的赋值语句就出错了，因为final引用是没办法二次赋值的。

所以Java里面的闭包，对外部变量只能读，不能写（但是可以修改这个外部变量的属性）。



### 4 总结

Kotlin的闭包设计更强大。
