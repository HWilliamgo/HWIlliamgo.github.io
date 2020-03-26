>来自慕课网的课程，自己做的笔记：http://www.imooc.com/video/13672

#1. 每一个app在手机上是有最大内存分配限制的，随着设备的不同而不同：
```
        textView=findViewById(R.id.tv);
        ActivityManager activityManager= (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        int memoryClass=activityManager.getMemoryClass();
        int largeMC=activityManager.getLargeMemoryClass();
        textView.setText("memoryClass: "+ memoryClass+"\n"+"largeMC: "+largeMC);
```
![](http://upload-images.jianshu.io/upload_images/7177220-60a9ad7af9992be2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而我在另一台虚拟机上面测试的一个memoryClass是180多M，largeMemoryClass是512M。

#2. 吃手机内存的大户：图片
#3. app切换时后台清理机制
 app切换时的LRU cache:
LRU算法：最近使用的app排在最前面，最少的可能被清理掉

#4. 查看内存情况的工具：
Tools-Android-Android Device Monitor
![](http://upload-images.jianshu.io/upload_images/7177220-05189c2d72d6932c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

左边选住你的进程，然后点update heap
![](http://upload-images.jianshu.io/upload_images/7177220-211fd800db9c7347.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点一下Cause GC,出现数据
![image.png](http://upload-images.jianshu.io/upload_images/7177220-bd643aac527f4d09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
heap size是系统分配的内存，allocated是已占用的内存,free是还没占用的内存。
data object是对象，class object是类,一般开发就看这两个，如果数值一直在增加，说明可能有内存泄漏，一般都是稳定在一个数值。

#5. 内存优化小技巧：
* 用StringBuilder替换+来拼接字符串，速度提升上千倍，（因为+号的内部处理也是用StringBuider来append，但是用很多加号就会new很多StringBuilder，就会凭空生成很多对象，降低效率）
* 用ArrayMap、SparseArray替换HashMap
* 内存抖动：在Anroid Profiler的memory那里看到内存一会高一会低，是因为方法中突然new出大量的对象（比如for循环里面new对象），然后方法结束GC把对象回收，一下载内存减少。
这样频繁调用GC对app流畅性影响非常大（好像是GC调用的时候所有线程暂停）
#6.  
再小的class都会耗费0.5KB
HashMap一个entry需要额外占用32B

#7. 对象复用
* 复用系统自带的资源
* ListView,GridView的ConvertView的复用
* 避免在自定义View的onDraw里面创建对象，因为只要一个View的状态改变，就会执行onDraw，在那里创建对象会影响整体UI绘制，造成界面卡顿。

#8. 内存泄漏
* 内存泄漏会导致可用的Heap越来越少，然后频繁调用GC，最后还可能会OOM（out of memory）
* 注意Activty里面的内存泄漏，比如在activity里面的开一个内部类Mythread,内部类默认对外部类有一个引用，因此不要再内部类里面去处理耗时操作，网络请求之类的。
-----解决方法：耗时操作放去Service
* 用Application的context而不是activty的context;
* Cursor对象是否及时关闭，一般涉及到数据库操作，用完了记得关闭。
