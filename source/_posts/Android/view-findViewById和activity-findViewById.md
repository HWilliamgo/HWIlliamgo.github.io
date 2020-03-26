直接内容见我的stackoverflow上的提问：
[android view.findViewById and activity.findViewById, one cannot show the data](https://stackoverflow.com/questions/47131227/android-view-findviewbyid-and-activity-findviewbyid-one-cannot-show-the-data)


以下内容大多为上面连接的直接截图：

![](http://upload-images.jianshu.io/upload_images/7177220-0b95652a00ebfb2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/7177220-c56d56682be44c9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*以上是我问的问题*

下面是两个优质回答
1、
![](http://upload-images.jianshu.io/upload_images/7177220-c2dc52c4135ad30f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、

![](http://upload-images.jianshu.io/upload_images/7177220-c3d3c1df416a95b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**总结**
>View.findViewById行不通是因为当前的view没有和activity绑定，少了一个activity.setContentView，而在onCreate里可以直接用也是因为调用了setContentView进行了绑定，所以要用的话就直接activity.findViewById.
