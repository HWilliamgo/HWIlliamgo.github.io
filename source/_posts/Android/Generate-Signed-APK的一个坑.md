>给这玩意坑了好长时间，不过还好后面还是解决了。

 入口：

![](http://upload-images.jianshu.io/upload_images/7177220-3072c2fc6dd43415.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
进入：

![](http://upload-images.jianshu.io/upload_images/7177220-9f03d8fca78d40da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下一步：

![](http://upload-images.jianshu.io/upload_images/7177220-b43e55eb704797b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>这个地方注意了，V2的签名方式是Android 7.0之后的签名方式，顾名思义：需要运行在7.0以后的手机上才能使用这种签名方式，但是如果不是android7.0的手机我们用V2的签名方式签名了会啥情况？

很蛋疼

![](http://upload-images.jianshu.io/upload_images/7177220-6dd845ef22e95bbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 是的是的，就是找不到签名证书了。

* 结论：
避免不适配咋办？直接用V1的签名就行了呀。
或者V1和V2都用，也没问题。
