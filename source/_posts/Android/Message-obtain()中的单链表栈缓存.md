# Message.obtain()中的单链表栈缓存

Android中的Message.java用单链表实现了一个size=50的栈，用作缓存。以下结合源码和图分析存取过程。

### 存

``` java
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

**调用时机**：在Looper.loop中调用，具体地说在Handler.handleMessage()之后立刻调用。

``` java
public void static loop(){
    final Looper me=Looper.myLopper();
    final MessageQueue queue = me.mQueue;
    for(;;){
        //...
        Message msg = queue.next();
        msg.target.dispatchMessage(msg);
        //上面的dispatchMessage最终执行到handler的handleMessasge了
        msg.recycleUnchecked();
    }
}
```



### 取

``` java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

**调用时机：**这个应该由开发者在需要Message对象的时候手动调用来获取缓存池中的Message对象。



### 结合图来分析存取过程中单链表栈的变化

#### 第一次存取

1. 初始状态，sPool指向null。没有任何Messsage对象。

![1547627128889.png](https://upload-images.jianshu.io/upload_images/7177220-0681293feda3b545.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 调用一次Message.obtain()。由于sPool==null，因此直接new Message()并返回给开发者使用，在用完后，Looper为其调用了 msg.recycleUnchecked();将其回收。

   ``` java
   synchronized (sPoolSync) {
       if (sPoolSize < MAX_POOL_SIZE) {
           next = sPool；//1
           sPool = this;//2
           sPoolSize++;//3
       }
   }
   ```

   开始分析，此时还未执行代码段1,2,3。

 ![1547627414730.png](https://upload-images.jianshu.io/upload_images/7177220-2e803375a8817c6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   执行1。（此时Message1.next实际上指向了Null）
![](https://upload-images.jianshu.io/upload_images/7177220-32a6ac31cdecf1a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   执行2。（sPool引用指向了对象Message1）
![](https://upload-images.jianshu.io/upload_images/7177220-6713a7ca06edb9d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


   执行3。将缓存池数量标记+1。

#### 第二次存取

1. 初始状态

![](https://upload-images.jianshu.io/upload_images/7177220-6713a7ca06edb9d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 开发者调用Messsage.obatin()来获取Message对象

   ``` java
   public static Message obtain() {
       synchronized (sPoolSync) {
           if (sPool != null) {
               Message m = sPool;//1
               sPool = m.next;//2
               m.next = null;//3
               m.flags = 0; // clear in-use flag
               sPoolSize--;
               return m;//4
           }
       }
       return new Message();
   }
   ```

   执行1。（创建一个Message m引用，指向了sPool所指向的对象Message1）
![1547628442298.png](https://upload-images.jianshu.io/upload_images/7177220-ead30088c7f79264.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



   执行2。（让sPool指向null）
![1547628468565.png](https://upload-images.jianshu.io/upload_images/7177220-b7b1f14b8e3b0dd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




   执行3。还是上面的原图

   执行4。将引用m返回，交给开发者，其指向了对象Message1，开发者可以拿着这个引用对其做处理。

   

3. 由Looper来调用msg.recycleUnchecked将其回收。其所有过程和第一次存取的回收过程如出一辙。

   
#### 多次存取

1. 一次性调用多次obtain，由于当缓存池中没有Message对象时，sPool指针就会==null，因此就会直接生成Message实例，如图：

 ![1547628887922.png](https://upload-images.jianshu.io/upload_images/7177220-3779d68feb0c0c1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 此时每一个Message都被handler处理完成，并准备一个一个被Looper回收。

   执行Message1的回收：

 ![1547628987637.png](https://upload-images.jianshu.io/upload_images/7177220-27506b615819d57b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   执行Message2的回收：

  ![1547629051003.png](https://upload-images.jianshu.io/upload_images/7177220-9ef3dfdc1be898e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


   执行Message3的回收
 ![1547629167436.png](https://upload-images.jianshu.io/upload_images/7177220-87f8263db4bbcf96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
   执行Message4的回收
![1547629182292.png](https://upload-images.jianshu.io/upload_images/7177220-08e7664e714d1744.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**如图，此时Message维护了一个单链表构成的栈，sPool指向栈顶**

当obtainMessage()时，创建一个引用指向sPool，即指向栈顶的Message4，将sPool指向Message4.next即Message3，再将Message4.next=null，将Message4对象脱离单链!
表，最后将引用返回给开发者，供开发者操纵对象Message4。图不再给出。



