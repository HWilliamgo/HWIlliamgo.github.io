# ThreadLocal

> `ThreadLocal`是一个线程独立存储类，通过他的`get`和`set`方法，在不同的线程中，可以独立地存取不同的`value`。
>
> 
>
> 每次回顾`Looper`源码的时候，都会忘了`ThreadLocal`是如何实现和工作的，故这次记录下来。

[TOC]

## 工作原理图

![](https://upload-images.jianshu.io/upload_images/7177220-1d4fc68a677a2449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ThreadLocal中的ThreadLocalMap

![](https://upload-images.jianshu.io/upload_images/7177220-bb29144bb538d057.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先观察`ThreadLocal`的结构，有两个静态内部类，其中第二个`ThreadLocalMap`是他工作原理的最重要的类。

此时点开`Thread`类源码：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

会发现`Thread`引用了一个`ThreadLocal.ThreadLocalMap`，并且命名为`threadLocals`。并且这个引用在每一个`Thread`对象中只有一个。这样的设计使得每个`Thread`对象都有一个自己的`Map`。



### Entry extends WeakRefernce

再回到`ThreadLocalMap`中。![](https://upload-images.jianshu.io/upload_images/7177220-11f49c53a0f9900e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果看过`HashMap`或者`LinkedHashMap`源码应该会对这个`Entry`的设计很熟悉。代码为：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
    //...
    
    private Entry[] table;//Entry数组
    
    //set方法
    private void set(ThreadLocal<?> key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();
            if (k == key) {
                e.value = value;
                return;
            }
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
    //get方法
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }
}
```

与`HashMap`不同的是，

1. 他继承了`WeakRefernce`，`key`必须是`ThreadLocal`，`value`是Object，即任意类型，并且在构造函数中将`key`传入作为弱引用构造函数的参数。
2. `Entry`并没有引用一个`next`之类的指针，因此无法构成`HashMap`那样的链表。
3. 纯粹的是维护一个`Entry`数组。

继承`WeakReferenc`是因为：`ThreadLocal`作为key，如果是强引用，会一直被这个`ThreadLocalMap`引用，而只要后者对应的线程没有结束，那么就算在别处因为不需要了，而将该`ThreadLoacal`释放(=null)，由于在map中的强引用，`ThreadLocal`对象也无法回收。

因此这里只要设计成弱引用，在别处释放后，垃圾收集器就可以顺利回收`ThreadLocal`对象。



## ThreadLocal

ThreadLocal对象实质上是做为一个key，在不同的线程中的map里，存取不同的对象。

```java
public ThreadLocal() {
    //...
    
	public T get() {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);//当前对象作为key去getEntry(this)
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    public void set(T value) {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);//当前对象作为key去set(this,value)
        else
            createMap(t, value);
    }
}
```



---

剩下的就是一些实现细节了，直接到源码里面去看就行了，主要还是看看顶部的工作原理图，理解原理就行。








