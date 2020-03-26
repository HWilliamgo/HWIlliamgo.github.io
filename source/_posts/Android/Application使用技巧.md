>这里简单介绍application类的使用技巧
* 首先，写一个类来继承自Application,然后重写onCreate方法：

![](http://upload-images.jianshu.io/upload_images/7177220-39f255abe2b7ae1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后在Manifest文件里面把我们自己写的Application配进去：

![](http://upload-images.jianshu.io/upload_images/7177220-b0ed6a4607a8c185.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>这里注意了，此前我自己也遇到过这个麻烦，那么就是在用这个MyApplication的时候，是绝对不能这么写的：
```
MyApplication app=new MyApplication();
我当时这么写的结果就是，代码都不知道怎么错的，调试了半天，
还以为是我数据库初始化出了问题。这个很严重。错的原因是，
Application这个东西是Android系统的一个类，是在应用打开的时候
有且只有初始化一次，我们自己在代码里面创建的那已经不是
Application了，是啥我也不懂，因此直接new一个Application是
绝对不行的，用法是错的。
```
#正确做法如下：
我们采取单例模式，这样application就可以不用创建多次，复用一个对象就可以了。

![](http://upload-images.jianshu.io/upload_images/7177220-38e2c11226645a79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* OK大功告成，现在任何的类都可以获取到我们的context了。

![](http://upload-images.jianshu.io/upload_images/7177220-7000bc6cc9c953f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
