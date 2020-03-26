>借鉴自http://www.cnblogs.com/Claire6649/p/5941145.html
1. 在drawable中创建selector:
```
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_checked="true" android:drawable="@mipmap/check" />
    <item android:state_checked="false" android:drawable="@mipmap/uncheck" />
    <item android:drawable="@mipmap/uncheck" />
</selector>
```
2. 在values文件夹下的styles.xml文件中添加CustomCheckboxTheme样式。
```
<style name="CustomCheckBoxTheme" parent="Widget.AppCompat.CompoundButton.CheckBox">
        <item name="android:button">@drawable/check_box_selector</item>        
</style>
```


# 说明
* 在drawable中引用mipmap的问题
mipmaps 是一种可视化技术渲染算法，速度相当快，所以google强烈建议使用mipmap装图片。把图片放到mipmaps可以提高系统渲染图片的速度，提高图片质量，减少GPU压力。

***
然而问题是我们在drawable中引用mipmap资源的时候是没有代码提示的，正确的做法就是直接自己动手打吧

![你看，这就没有代码提示](http://upload-images.jianshu.io/upload_images/7177220-e182f5763fccadd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是当我们直接手动输入之后的样子：

![不会报错了](http://upload-images.jianshu.io/upload_images/7177220-7e3de8748a11964c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 所以解决方法是：没有提示，自己敲出来
