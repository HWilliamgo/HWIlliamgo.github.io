
## 本文描述的内容如题：重载的陷阱，静态分派，重写，动态分派
``` java
public class StaticDispatch {
    static abstract class Human{}
    static class Man extends Human{}
    static class Woman extends Human{}

    public void sayHello(Human guy){
        System.out.println("hello,guy!");
    }
    public void sayHello(Man guy){
        System.out.println("hello,gentleman!");
    }
    public void sayHello(Woman guy){
        System.out.println("hello,lady!");
    }

    public static void main(String[] args){
        Human man=new Man();
        Human woman=new Woman();

        StaticDispatch sr=new StaticDispatch();

        sr.sayHello(man);
        sr.sayHello(woman);
    }
}

//print:
//hello,guy!
//hello,guy!
```
我刚看代码的时候，是以为结果会是`hello,gentleman(换行)hello,lady!`这样的，但是为什么选择了参数为`Human`的重载方法呢？
要点总结如下：
1. 在`Human man=new Man()`中，`Human`被称为**静态类型或外观类型**，`Man`被称为**实际类型**。
2. 编译器在编译程序的时候并不知道一个对象的实际类型是什么，只能在运行期才可确定；而静态类型是在编译器可知的。
3. 上述代码中可以定义了两个静态类型相同但是实际类型不同的变量，但是虚拟机（或者说编译器）在重载时是通过参数的**静态类型**而不是**实际类型**作为判定依据。
4. 因此，在编译阶段，Javac编译器会根据参数的**静态类型**决定使用哪个重载版本，所以选择了`sayHello(Human）`作为调用目标。
5. 所有依赖**静态类型**来**定位**方法执行版本的分派动作称为**静态分派**，静态分派的典型应用是方法重载。

> 结论：确定**重载哪个方法**通过对象的**静态类型**。

---

### 上述描述了静态分派，下文描述动态分派：
动态分派和重写有很密切的关联：

``` java
public class Main {
    public static void main(String args[]) {
        Human man=new Man();
        Human woman=new Woman();

        man.sayHello();
        woman.sayHello();

        man=new Woman();
        man.sayHello();
    }
    static abstract class Human{
        protected abstract void sayHello();
    }
    static class  Man extends Human{
        @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }
    static class Woman extends Human{
        @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }
}
/*
打印:
man say hello
woman say hello
woman say hello
 */
```
这个打印结果是毫无悬念且理所当然的。
很明显，这里打印出不同的结果是因为对象的实际类型不同。具体的分析和解释由于我自己也不是很明白，因此推荐直接查看《深入理解java虚拟机》P253的内容。
> 结论：**重写的方法**的调用，是根据**运行期**确定出的**对象的实际类型**，来确定具体调用哪个方法的。


## 总结：在实际的开发中，就是方法的重载那里要注意一下，是根据对象的静态类型来选择要重载哪个方法，而方法的重写则和平时没有什么区别。
