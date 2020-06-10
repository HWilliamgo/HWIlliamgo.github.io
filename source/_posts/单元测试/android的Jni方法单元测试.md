---
title: android的Jni方法单元测试
tags:
  - android单元测试
index_img: 'https://gitee.com/HWilliamgo/pictures/raw/master/img/wallhaven-2eq3gg.jpg'
banner_img: 'https://gitee.com/HWilliamgo/pictures/raw/master/img/wallhaven-2eq3gg.jpg'
categories:
  - 单元测试
date: 2020-06-06 17:23:25
---

> 本文记录：
>
> 使用cmake来构建android jni的c层代码。并使用android instrumented unit test 来测试jni方法和c层方法。



### 1 概述

1. jni方法为什么要用单元测试？

   有的jni方法在项目里触发条件复杂，作为开发者需要像普通用户那样去操作触发对应逻辑才能测试到Jni方法，效率低下。而使用单元测试则可以比较高效率地测试比较难触发到调用jni的场景。

2. 什么场景下用单元测试来测试jni方法？

   理论上来说所有的代码都能够在单元测试的环境下被测试到，但是有时需要一些比较复杂的上下文，或者代码的上下文搭建不容易，而直接运行项目测试更简单，所以，不是所有的jni方法都适合用单元测试。例如你要测试一些访问网络的逻辑，但是网络请求的代码没有封装，而是直接放在Activity或者Fragment中和UI逻辑耦合了，那么这个时候进行单元测试就比较麻烦了，需要你把网络请求逻辑从UI层抽取出来。jni方法

### 2 如何操作

#### 2.1 为某个类或者方法生成对应的测试文件。

![image-20200608091601281](https://gitee.com/HWilliamgo/pictures/raw/master/img/image-20200608091601281.png)

对想要测试的jni类进行generate操作，然后有一个Test...选项。点击。

![image-20200608091752660](https://gitee.com/HWilliamgo/pictures/raw/master/img/image-20200608091752660.png)

此时生成的NDKToolsTest类如下图所示：

![image-20200609190718766](https://gitee.com/HWilliamgo/pictures/raw/master/img/image-20200609190718766.png)

每个生成的方法和类都有一个run按钮。

点击类上的run，则会运行该类所有的测试方法。点击方法上的run，则只运行该方法。

#### 2.2 build产物

当我们运行单元测试类的时候，我们查看build目录：

``` 
在目录~/app/build/outputs/apk下
.    
├── androidTest
│   └── debug
│       ├── app-debug-androidTest.apk
│       └── output.json
└── debug
    ├── app-debug.apk
    └── output.json

```

分别生成了两个apk，我们来分析一下这两个APK。



首先看androidTest.apk

![image-20200609232819612](https://gitee.com/HWilliamgo/pictures/raw/master/img/image-20200609232819612.png)

这个apk很明显是单元测试的目录下的类文件编译出来的class文件，并包含了单元测试引用的一些类文件，例如androidx，android，单元测试中我用来打印的LogUtils等。

但是这个apk中很明显是没有so库的。那么so库应该就在另一个apk里了。

![image-20200609233134820](https://gitee.com/HWilliamgo/pictures/raw/master/img/image-20200609233134820.png)

这个apk的结构和输出的内容和我们普通编译的debug包没什么区别，那么我们看看当我们运行一个单元测试时，gradle执行了哪些task。

#### 2.3 gradle执行了哪些task

直接运行一个单元测试方法，可以在Build Output中看到执行了哪些task:

```
Executing tasks: [:app:assembleDebug, :app:assembleDebugAndroidTest] in project /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake

ABIs [arm64-v8a,armeabi-v7a,armeabi] set by 'android.injected.build.abi' gradle flag contained 'ARMEABI, ARM64_V8A' not targeted by this project.
ABIs [arm64-v8a,armeabi-v7a,armeabi] set by 'android.injected.build.abi' gradle flag contained 'ARMEABI, ARM64_V8A' not targeted by this project.
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :ffmpegprebuild:preBuild UP-TO-DATE
> Task :ffmpegprebuild:preDebugBuild UP-TO-DATE
> Task :lame:preBuild UP-TO-DATE
> Task :lame:preDebugBuild UP-TO-DATE
> Task :ffmpegprebuild:packageDebugRenderscript NO-SOURCE
> Task :lame:packageDebugRenderscript NO-SOURCE
> Task :app:checkDebugManifest
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :ffmpegprebuild:compileDebugAidl NO-SOURCE
> Task :app:generateDebugBuildConfig
> Task :app:mainApkListPersistenceDebug
> Task :app:generateDebugResValues
> Task :app:generateDebugResources
> Task :lame:compileDebugAidl NO-SOURCE
> Task :app:compileDebugAidl NO-SOURCE
> Task :ffmpegprebuild:generateDebugResValues
> Task :ffmpegprebuild:compileDebugRenderscript NO-SOURCE
> Task :ffmpegprebuild:generateDebugResources
> Task :ffmpegprebuild:packageDebugResources
> Task :lame:generateDebugResValues
> Task :lame:compileDebugRenderscript NO-SOURCE
> Task :lame:generateDebugResources
> Task :lame:packageDebugResources
> Task :app:createDebugCompatibleScreenManifests
> Task :ffmpegprebuild:checkDebugManifest
> Task :lame:checkDebugManifest
> Task :ffmpegprebuild:processDebugManifest
> Task :lame:processDebugManifest
> Task :app:processDebugManifest
> Task :ffmpegprebuild:parseDebugLibraryResources
> Task :app:mergeDebugResources
> Task :ffmpegprebuild:generateDebugBuildConfig
> Task :lame:parseDebugLibraryResources
> Task :ffmpegprebuild:generateDebugRFile
> Task :ffmpegprebuild:compileDebugKotlin
> Task :lame:generateDebugBuildConfig
> Task :ffmpegprebuild:javaPreCompileDebug
> Task :ffmpegprebuild:compileDebugJavaWithJavac
> Task :lame:generateDebugRFile
> Task :app:processDebugResources
> Task :lame:compileDebugKotlin
> Task :ffmpegprebuild:bundleLibCompileDebug
> Task :lame:javaPreCompileDebug
> Task :lame:compileDebugJavaWithJavac
> Task :app:generateJsonModelDebug UP-TO-DATE
> Task :lame:bundleLibCompileDebug

> Task :app:compileDebugKotlin
w: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/main/java/com/hwilliam/jnilearncmake/MainActivity.kt: (38, 13): Variable 'pcmFilePath' is never used
w: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/main/java/com/hwilliam/jnilearncmake/MainActivity.kt: (39, 13): Variable 'mp3FilePath' is never used
w: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/main/java/com/hwilliam/jnilearncmake/SimplePcmPlayer.kt: (75, 22): 'constructor AudioTrack(Int, Int, Int, Int, Int, Int)' is deprecated. Deprecated in Java

> Task :ffmpegprebuild:generateJsonModelDebug UP-TO-DATE

> Task :ffmpegprebuild:externalNativeBuildDebug
Build ff_armeabi-v7a
[1/2] Building C object CMakeFiles/ff.dir/ff.c.o
[2/2] Linking C shared library /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/build/intermediates/cmake/debug/obj/armeabi-v7a/libff.so

> Task :ffmpegprebuild:mergeDebugJniLibFolders
> Task :lame:mergeDebugJniLibFolders
> Task :app:javaPreCompileDebug
> Task :app:compileDebugJavaWithJavac
> Task :ffmpegprebuild:mergeDebugNativeLibs
> Task :ffmpegprebuild:stripDebugDebugSymbols
> Task :ffmpegprebuild:transformNativeLibsWithIntermediateJniLibsForDebug
> Task :lame:mergeDebugNativeLibs
> Task :lame:stripDebugDebugSymbols
> Task :lame:transformNativeLibsWithIntermediateJniLibsForDebug

> Task :app:externalNativeBuildDebug
Build native-lib_armeabi-v7a
[1/3] Building C object CMakeFiles/native-lib.dir/native-lib.c.o
[2/3] Building C object CMakeFiles/native-lib.dir/libB.c.o
/Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/main/cpp/native-lib.c:100:9: warning: array index 10 is past the end of the array (which contains 5 elements) [-Warray-bounds]
    y = a[10];
        ^ ~~
/Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/main/cpp/native-lib.c:98:5: note: array 'a' declared here
    int a[5];
    ^
1 warning generated.
[3/3] Linking C shared library /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/build/intermediates/cmake/debug/obj/armeabi-v7a/libnative-lib.so

> Task :app:compileDebugSources
> Task :app:mergeDebugShaders
> Task :app:compileDebugShaders
> Task :app:generateDebugAssets
> Task :ffmpegprebuild:mergeDebugShaders
> Task :ffmpegprebuild:compileDebugShaders
> Task :ffmpegprebuild:generateDebugAssets
> Task :ffmpegprebuild:packageDebugAssets
> Task :lame:mergeDebugShaders
> Task :lame:compileDebugShaders
> Task :lame:generateDebugAssets
> Task :lame:packageDebugAssets
> Task :app:mergeDebugAssets
> Task :app:processDebugJavaRes NO-SOURCE
> Task :ffmpegprebuild:processDebugJavaRes NO-SOURCE
> Task :lame:processDebugJavaRes NO-SOURCE
> Task :ffmpegprebuild:bundleLibResDebug
> Task :lame:bundleLibResDebug
> Task :ffmpegprebuild:bundleLibRuntimeDebug
> Task :ffmpegprebuild:createFullJarDebug
> Task :lame:bundleLibRuntimeDebug
> Task :lame:createFullJarDebug
> Task :app:checkDebugDuplicateClasses
> Task :app:transformClassesWithDexBuilderForDebug
> Task :app:validateSigningDebug
> Task :app:signingConfigWriterDebug
> Task :app:mergeDebugJniLibFolders
> Task :app:preDebugAndroidTestBuild SKIPPED
> Task :app:compileDebugAndroidTestAidl NO-SOURCE
> Task :app:processDebugAndroidTestManifest
> Task :app:compileDebugAndroidTestRenderscript NO-SOURCE
> Task :app:generateDebugAndroidTestBuildConfig
> Task :app:mainApkListPersistenceDebugAndroidTest
> Task :app:generateDebugAndroidTestResValues
> Task :app:generateDebugAndroidTestResources
> Task :app:mergeDebugAndroidTestResources
> Task :app:mergeDebugAndroidTestShaders
> Task :app:compileDebugAndroidTestShaders
> Task :app:generateDebugAndroidTestAssets
> Task :app:mergeDebugAndroidTestAssets
> Task :app:bundleDebugClasses
> Task :app:processDebugAndroidTestJavaRes NO-SOURCE
> Task :app:mergeDebugAndroidTestJniLibFolders
> Task :app:checkDebugAndroidTestDuplicateClasses
> Task :app:validateSigningDebugAndroidTest
> Task :app:signingConfigWriterDebugAndroidTest
> Task :app:mergeDebugAndroidTestNativeLibs
> Task :app:mergeDebugNativeLibs
> Task :app:stripDebugDebugSymbols
> Task :app:mergeDebugJavaResource
> Task :app:processDebugAndroidTestResources

> Task :app:compileDebugAndroidTestKotlin
w: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/androidTest/java/com/hwilliam/jnilearncmake/ExampleInstrumentedTest.kt: (4, 29): 'AndroidJUnit4' is deprecated. Deprecated in Java
w: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/app/src/androidTest/java/com/hwilliam/jnilearncmake/ExampleInstrumentedTest.kt: (16, 10): 'AndroidJUnit4' is deprecated. Deprecated in Java

> Task :app:mergeExtDexDebugAndroidTest
> Task :app:javaPreCompileDebugAndroidTest
> Task :app:compileDebugAndroidTestJavaWithJavac
> Task :app:compileDebugAndroidTestSources
> Task :app:transformClassesWithDexBuilderForDebugAndroidTest
> Task :app:mergeExtDexDebug
> Task :app:mergeDebugAndroidTestJavaResource
> Task :app:mergeDexDebugAndroidTest
> Task :app:packageDebugAndroidTest
> Task :app:assembleDebugAndroidTest
> Task :app:mergeDexDebug
> Task :app:packageDebug
> Task :app:assembleDebug

Deprecated Gradle features were used in this build, making it incompatible with Gradle 6.0.
Use '--warning-mode all' to show the individual deprecation warnings.
See https://docs.gradle.org/5.4.1/userguide/command_line_interface.html#sec:command_line_warnings

BUILD SUCCESSFUL in 10s
94 actionable tasks: 92 executed, 2 up-to-date

```

通过第一行的：

```
Executing tasks: [:app:assembleDebug, :app:assembleDebugAndroidTest] in project ...
```

可以看到单元测试会通过两个task来构建产物：

`assembleDebug`和`assembleDebugAndroidTest`

而他们则分别构建出了：**app-debug.apk**和**app-debug-androidTest.apk**。

该文章则描述了TestRunner通过一定权限，可以直接调用被测试APK的类、方法、资源，以及向其发送用户事件。：[Android测试初探（一） Android测试技术简介和原理浅析]([https://www.paincker.com/android-test-1#%E4%BE%8B%E4%B8%80](https://www.paincker.com/android-test-1#例一)) 

### 3 总结

通过android单元测试，可以简化某些特定复杂逻辑下的测试过程，减少测试和调试的时间。

