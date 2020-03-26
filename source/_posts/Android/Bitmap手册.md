> 本篇参考大量文章学习总结：


[深入理解bitmap](http://blog.csdn.net/xxxzhi/article/details/51607765)
[郭霖： Android高效加载大图、多图解决方案，有效避免程序OOM](http://blog.csdn.net/guolin_blog/article/details/9316683)
[玩转Android Bitmap](http://www.jianshu.com/p/3950665e93e6)

>内容：
1.bitmap实现内存优化
2.bitmap和BitmapFactory各参数讲解

实现效果：一张原图从占内存6M多削减到占内存0.2M左右


##1. 优化内存
上代码先：

* 首先是decodeBitmapFraomResource(),参数顾名思义。
![](http://upload-images.jianshu.io/upload_images/7177220-85d3e76e589057ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 最关键部分的算法，通过比较原图宽高和我们要求的宽高来取得缩放比例。
![](http://upload-images.jianshu.io/upload_images/7177220-1cef4e72b64a4cd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
用的时候：

```
Bitmap bitmap=decodeBitmapFromResource(getResources(),R.drawable.picture
                ,100,100);
imageView.setImageBitmap(bitmap);
```
非常强势，自动缩放成我们定义的100*100的尺寸要求。

接下来进行对比：

*  不进行优化：
![](http://upload-images.jianshu.io/upload_images/7177220-91f6c1d699cbbc87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
打印：
![](http://upload-images.jianshu.io/upload_images/7177220-0f7411cd00258478.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是这张图片占用了6895876 Bytes=6.8MB内存

* 进行优化：
用上面刚写的算法来搞：
![](http://upload-images.jianshu.io/upload_images/7177220-07c733ff4f552a04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打印：
![](http://upload-images.jianshu.io/upload_images/7177220-eab2280586970813.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

276676 Bytes=0.27MB
缩小了25倍。

**强势的一匹**

##2. Bitmap和BitmapFactory各参数讲解:
1.创建bitmap：
* Bitmap的静态方法`createBitmap()
![](http://upload-images.jianshu.io/upload_images/7177220-2b8ba2ababe4e34d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)
* BitmapFactory的`decode`系列静态方法
![](http://upload-images.jianshu.io/upload_images/7177220-2f2027629a8f5afa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)

2 .Config:
![](http://upload-images.jianshu.io/upload_images/7177220-ec4592240530460d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有四个参数：
* Bitmap.Config.ALPHA_8：颜色信息只由透明度组成，占8位。
* Bitmap.Config.ARGB_4444：颜色信息由透明度与R（Red），G（Green），B（Blue）四部分组成，每个部分都占4位，总共占16位。
* Bitmap.Config.ARGB_8888：颜色信息由透明度与R（Red），G（Green），B（Blue）四部分组成，每个部分都占8位，总共占32位。是Bitmap默认的颜色配置信息，也是最占空间的一种配置。
* Bitmap.Config.RGB_565：颜色信息由R（Red），G（Green），B（Blue）三部分组成，R占5位，G占6位，B占5位，总共占16位。

***通常我们优化Bitmap时，当需要做性能优化或者防止OOM（Out Of Memory），我们通常会使用Bitmap.Config.RGB_565这个配置，因为Bitmap.Config.ALPHA_8只有透明度，显示一般图片没有意义，Bitmap.Config.ARGB_4444显示图片不清楚，Bitmap.Config.ARGB_8888占用内存最多。***
>他们的工作原理：
```
int b = 1;
switch (bitmap.getConfig()) {
    case ALPHA_8:
        b = 1;
        break;
    case ARGB_4444:
        b = 2;
        break;
    case ARGB_8888:
        b = 4;
        break;
}
int bytes1 = bitmap.getWidth() * bitmap.getHeight() * b;
int bytes2 = bitmap.getByteCount();　
//从api12才有的接口
//bytes=bytes2;
```

