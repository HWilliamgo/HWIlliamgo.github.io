

### 1. 常见用法

#### 1. 在ext这个map中放字符串或者基本数据类型

在android的rootProject的build.gradle中，定义如下代码块

``` groovy
ext {
    compileSdkVersion = 25
    buildToolsVersion = "26.0.0"
    minSdkVersion = 14
    targetSdkVersion = 22
    appcompatV7 = "com.android.support:appcompat-v7:$androidSupportVersion"
}
```

然后在app模块下，通过

```groovy
rootProject.ext.compileSdkVersion
rootProject.ext.buildToolsVersion
```

这种方式来访问在根目录build.gradle下定义的变量。



#### 2. 在ext这个map中再map

新建一个config.gradle文件

在里面填充

```groovy
ext {
	//创建了一个名为android的，类型为map的变量，groovy中可以用[]来创建map类型。那么就是一个map下面又创     
    //建了一个map，且名字叫做android。
    android = [
            compileSdkVersion: 23,
            buildToolsVersion: "23.0.2",
            minSdkVersion    : 14,
            targetSdkVersion : 22,
    ]
    dependencies = [
            appcompatV7': 'com.android.support:appcompat-v7:23.2.1',
            design      : 'com.android.support:design:23.2.1'
    ]
}
```

然后在根目录的build.gradle的头部应用这个脚本：

``` groovy
apply from: "config.gradle"
```

那么在app/build.gradle中，可以直接这样用：

```groovy
android {
    compileSdkVersion rootProject.ext.android.compileSdkVersion
    buildToolsVersion rootProject.ext.buildToolsVersion
    defaultConfig {
        applicationId "com.wuxiaolong.gradle4android"
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode 1
        versionName "1.0"
    }
  
...
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile rootProject.ext.dependencies.appcompatV7
    compile rootProject.ext.dependencies.design
}
```

#### 3. 在ext这个map中放入函数类型的变量

```groovy
//用{}来创建函数类型对象，即闭包，赋值给变量myMethod
ext.myMethod = { param1, param2 ->
    // 闭包体
}
```

比如在rootProject中创建，那么任何一个其他的project对象都可以通过下面的方式访问到这个方法。

```groovy
rootProject.ext.myMethod("1","2")
```



### 2. 是什么：

根据ext属性的[官方文档](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html#org.gradle.api.plugins.ExtraPropertiesExtension)，ext属性是[ExtensionAware](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtensionAware.html)类型的一个特殊的属性，本质是一个Map类型的变量，而

ExtentionAware接口的实现类为`Project`, `Settings`, `Task`, `SourceSet`等，ExtentionAware可以在运行时扩充属性，而这里的ext，就是里面的一个特殊的属性而已。如下图：

![](https://upload-images.jianshu.io/upload_images/7177220-41485d8af4ff7083.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于ExtensionAware的大概的解释：该接口的子类型为`Project`, `Settings`, `Task`, `SourceSet`，他们都有一个变量叫做`extensions`(我猜想该变量也是一个类似map的实现原理)，可以往该变量中放置属性。



### 3. 使用ext属性的优势

用ext属性，和直接在build.gradle中用def定义变量的好处是什么？



ext属性可以伴随对应的`ExtensionAware`对象在构建的过程中被其他对象访问，例如你在rootProject中声明的ext中添加的内容，就可以在任何能获取到rootProject的地方访问这些属性，而如果只在rootProject/build.gradle中用def来声明这些变量，那么这些变量除了在这个文件里面访问之外，其他任何地方都没办法访问。

这是因为在build.gradle中直接定义的属性，只是作为gradle构建的生命周期中的`Configuration`阶段的局部变量而已（我在[Gradle构建的生命周期和其对象的理解](https://www.jianshu.com/p/4f0ff9bd2f62)中猜想该阶段是在执行Project对象的构造函数），而往ext属性中写入变量，则可以在整个构建的生命周期都访问到那些变量。



此外要注意，ext属性是属于拥有他的相应的对象的，比如Project对象，因此只要能访问到对应的Project对象，就能访问到对应的ext属性





thanks:

http://wuxiaolong.me/2016/03/31/gradle4android2/

https://stackoverflow.com/questions/37186108/gradle-def-vs-ext

https://stackoverflow.com/questions/27777591/how-to-define-and-call-custom-methods-in-build-gradle?noredirect=1&lq=1
