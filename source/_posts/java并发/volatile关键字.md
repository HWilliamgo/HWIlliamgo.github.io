弄懂volatile之前首先确保弄懂了java内存模型，可参考我的整理
[Java内存模型](http://www.jianshu.com/p/bc777f741a2f)

volatile的作用如下：
1. 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
2. 禁止进行指令重排序。(涉及volatile变量的代码的前面和后面的代码不会进行重排序)。

注意：
1. volatile不保证被修饰的变量的操作的原子性（要保证操作的原子性建议使用同步:synchronized和Lock）。

使用场景：
* 标记状态量：
```
volatile boolean flag = false;
 
while(!flag){
    doSomething();
}
 
public void setFlag() {
    flag = true;
}
```
```
//保证了线程1的语句一一定在语句二之前执行（因为volatile禁止重排序），
// 否则可能会线程1的语句二先执行，这时线程2的while循环跳出，
// 立刻执行doSomethingWithConfig(context);
// 此时context还没赋值，出现空指针异常。
volatile boolean inited = false;
//线程1:
context = loadContext();  //语句一
inited = true;            //语句二
 
//线程2:
while(!inited ){
sleep()
}
doSomethingWithConfig(context);
```
* double check:
```
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {
         
    }
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```
参考：[Java并发编程：volatile关键字解析](http://www.cnblogs.com/dolphin0520/p/3920373.html)
