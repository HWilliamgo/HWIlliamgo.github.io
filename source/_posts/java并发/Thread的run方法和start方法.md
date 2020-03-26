结论：
* 调用Thread里的run方法的时候，会直接在主线程里通过对象调用该对象的run方法，就是普通的调用，不会开启线程。
* start方法会开启一个新的线程去执行run方法里的代码。
>代码：

![](http://upload-images.jianshu.io/upload_images/7177220-e091499fe623a94e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>结果：

![](http://upload-images.jianshu.io/upload_images/7177220-3a3b6da9d6b17873.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
