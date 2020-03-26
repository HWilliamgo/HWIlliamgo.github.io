今天遇到了一个需求是，要改应用的applicationId然后上架，那么我以前的做法是将应用的包名一起给改了，让包名和applicationId统一。但是我今天想了一下，是否可以不改包名，只改appId，那么后期就不用维护两套差异比较大的代码了，毕竟改了包名，包的结构会发生改变，那么git分支合并就会有比较多的冲突了。

**结论**：可以只改build.gradle中的applicationId来改包名，因为后者会覆盖掉Manifest文件中的<package>，但是有使用的限制场景（文末指出了限制场景）。



### 1. 理解Manifest中定义的包名和gradle中定义的applicationId的差异

#### 结论

在gradle构建apk的最后一步时，会将build.gradle中的applicationId，覆盖AndroidManifest.xml中定义的<package>，这样应用的ID就以gradle中指定的为准。而Manifest中指定的<package>，则影响的是真正的Java类目录，包括编译时的类目录和编译后生成的apk中的class文件的类路径。

#### 1.1 Manifest中的包名

##### 1.1.1 打包前后的表现

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.william.changepackage">

    <application
            android:allowBackup="true"
            android:icon="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:roundIcon="@mipmap/ic_launcher_round"
            android:supportsRtl="true"
            android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>

                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>

</manifest>
```

Manifest中的包名指的就是package=""中的内容，在创建一个项目的时候，就会指定这个值，并且这个值默认是和gradle中定义的applicationId是一致的。

``` groovy
android {
    //...
    defaultConfig {
        applicationId "com.william.changepackage"
 		//...
    }
    //...
}
```

那么看看此时的项目结构：

![](https://upload-images.jianshu.io/upload_images/7177220-1bc8484aec790c40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打一个debug包看看：

![](https://upload-images.jianshu.io/upload_images/7177220-5b88e27cc6d00233.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`BuildConfig`类和`R.java`类都是生成在`/包名/`路径下的。

而apk中，合并了的最终的Manifest:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    android:compileSdkVersion="28"
    android:compileSdkVersionCodename="9"
    package="com.william.changepackage"
    platformBuildVersionCode="28"
    platformBuildVersionName="9">
    <uses-sdk
        android:minSdkVersion="16"
        android:targetSdkVersion="28" />
    <application
        android:theme="@ref/0x7f0c0005"
        android:label="@ref/0x7f0b0027"
        android:icon="@ref/0x7f0a0000"
        android:debuggable="true"
        android:testOnly="true"
        android:allowBackup="true"
        android:supportsRtl="true"
        android:roundIcon="@ref/0x7f0a0001"
        android:appComponentFactory="android.support.v4.app.CoreComponentFactory">
        <activity
            android:name="com.william.changepackage.MainActivity">
            <intent-filter>
                <action
                    android:name="android.intent.action.MAIN" />
                <category
                    android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

注意，这里的`MainActivity`的声明，在编写的时候是没有加上包名前缀的，合并成最终的Manifest加上了，加的就是类路径，也就是Manifest合并前的<package>。

##### 1.1.2 存在的意义

Manifest中的<package>存在的意义是，`BuildConfig.java`，`R.java`等文件，都是在编译期生成的，那么这些文件生成了要放在哪里？放在我们的类路径中让我们的类能引用到`R.java`中定义的资源变量。那么`R.java`就需要被指明一个路径来生成，这就是<package>的意义，如果把<pakcage>随便写一个路径，然后打包，会遇到：

```
项目路径\changePackage\app\src\main\java\com\william\changepackage\MainActivity.kt: (10, 24): Unresolved reference: R
```

意思是`MainActivity.java`没办法引用到`R.java`了，因为`R.java`生成在了一个`MainActivity.java`无法识别的路径（即乱写的那个路径）。无法打包。

##### 1.1.3 书写规则

项目的类路径是什么，Manifest的<package>就怎么写，两边保持一致。

#### 1.2 gradle中的applicationId

applicationId就是用来表明应用ID的，就是应用市场上面的appId，应用区分于别的应用的唯一标志。

他与类路径无任何关系，你指定什么都不会影响类路径。

##### 1.2.1 完全修改applicationId

我把

``` groovy
android {
    //...
    defaultConfig {
        applicationId "com.william.changepackage"
 		//...
    }
    //...
}
```

改成了

``` groovy
android {
    //...
    defaultConfig {
        applicationId "how.are.you"
 		//...
    }
    //...
}
```

编译->打包->检查apk：

先看apk中的manifest文件：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    android:compileSdkVersion="28"
    android:compileSdkVersionCodename="9"
    package="how.are.you"
    platformBuildVersionCode="28"
    platformBuildVersionName="9">

    <uses-sdk
        android:minSdkVersion="16"
        android:targetSdkVersion="28" />

    <application
        android:theme="@ref/0x7f0c0005"
        android:label="@ref/0x7f0b0027"
        android:icon="@ref/0x7f0a0000"
        android:debuggable="true"
        android:testOnly="true"
        android:allowBackup="true"
        android:supportsRtl="true"
        android:roundIcon="@ref/0x7f0a0001"
        android:appComponentFactory="android.support.v4.app.CoreComponentFactory">

        <activity
            android:name="com.william.changepackage.MainActivity">

            <intent-filter>

                <action
                    android:name="android.intent.action.MAIN" />

                <category
                    android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>

```

1. <package>的确被覆盖成了how.are.you。
2. `MainActivity`的声明依然保留的是原本的类路径的前缀，而不是新的how.are.you.MainActivity。（改成新的了就会出问题了不是吗）



查看一下classes.dex

![](https://upload-images.jianshu.io/upload_images/7177220-319ce6452fe0af06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

类路径没有改变，依然是编译前类路径什么样现在类路径还是什么样。

##### 1.2.2 存在的意义

以前用Eclispse开发的时候，不用build.gradle，那时候的包名就等同于应用名。没有却别。

但是后来AndroidStudio开发，引入了gradle这个高级的构建系统，gradle支持applicationId，因为你可以这样操作：

``` groovy
android {
    defaultConfig {
        applicationId "com.example.myapp"
    }
    productFlavors {
        free {
            applicationIdSuffix ".free"
        }
        pro {
            applicationIdSuffix ".pro"
        }
    }
}
```

来对免费和付费版app发布不同的应用到应用商店。

或者用

``` groovy
android {
    ...
    buildTypes {
        debug {
            applicationIdSuffix ".debug"
        }
    }
}
```

这样debug和release包的appID不一样，那么就可以同时在一个设备装发布版和调试版应用了。

##### 1.2.3 书写规则

- 必须至少包含两段（一个或多个圆点）。
- 每段必须以字母开头。
- 所有字符必须为字母数字或下划线 [a-zA-Z0-9_]

### 2. applicationId的限制

Android文档中提示：

![](https://upload-images.jianshu.io/upload_images/7177220-6d225361d587e5e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我点进到[问题211768](https://issuetracker.google.com/issues/37102241)：

![](https://upload-images.jianshu.io/upload_images/7177220-a96b5f1885783872.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/7177220-4af40e2e13595575.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当WebView加载`file:///android_res/URLS`这种资源的时候，他的工作方式是，通过查找app的package name来找到R.java文件，从而引用res/下的资源，然而，当applicationId和package不同时，后者会在生成apk之前被覆盖掉，导致WebView查找不到这个资源，无法正常工作。

所幸，这个WebView的bug被部分地修复了，只要你的applicatIionId和包名拥有相同的前缀，那么就能被识别到。（我猜是他会从左到右根据"."分隔符去查找R.java资源文件）

所以，最保险的做法是，applicationId的前缀和包名完全相同。

如果要和包名不一样，那么就保证应用中的WebView不要用`file:///android_res/URLS`这种方式去引用资源，可以放在assets目录中。
