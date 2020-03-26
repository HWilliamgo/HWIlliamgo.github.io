1. 新建一个library
![](http://upload-images.jianshu.io/upload_images/7177220-f684af9859081854.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 添加项目android-maven插件
* 在project的gradle文件中
```
buildscript { 
  dependencies {
    classpath 'com.github.dcendents:android-maven-gradle-plugin:2.0' // Add this line
```
![图示project的gradle](http://upload-images.jianshu.io/upload_images/7177220-f65d942ca4882d7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 在Library的gradle文件中
```
 apply plugin: 'com.github.dcendents.android-maven'  

 group='com.github.YourUsername'
```
这里的YourUsername就是github的用户名。
![图示library的gradle](http://upload-images.jianshu.io/upload_images/7177220-c83e3fc84d7aea8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 接下来是上传到github:
![](http://upload-images.jianshu.io/upload_images/7177220-438ff0d5ff9e1fc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. push哪些文件上去呢？
官方实例：

![jitpack官方示例](http://upload-images.jianshu.io/upload_images/7177220-f67fdb63bb2e6d92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
说明：第一个是project的整个gradle文件，第二个是library的整个包，后面的都是project的东西，不要放application的包进来，那是多余的。
5. 生成release，就是发布版
![](http://upload-images.jianshu.io/upload_images/7177220-6b78a583086fb38f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点release，进去发布。
6. 进入jitPack官网进行最后的操作
https://jitpack.io/
![](http://upload-images.jianshu.io/upload_images/7177220-24d9b2bf089e347f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
把刚才的项目的github连接直接复制粘贴，他会自动搜索最近一次的release版，然后进行处理
![](http://upload-images.jianshu.io/upload_images/7177220-589489d43189ad1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Get it变成绿色的时候就是完成了，点击。
![](http://upload-images.jianshu.io/upload_images/7177220-ad0de8cad7ec2c72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
完成！

附上几个教程：
```简书里的中文教程```：
[Android 写自己的开源库，发布到 JitPack.io](https://www.jianshu.com/p/e443456bb506)
[AS快速上传Library到GitHub并通过JitPack打包集成](https://www.jianshu.com/p/b04ef4029b90)
```官方英文教程```：
[Publish an Android library](https://www.jitpack.io/docs/ANDROID/)
[Android-Example](https://github.com/jitpack/android-example)








