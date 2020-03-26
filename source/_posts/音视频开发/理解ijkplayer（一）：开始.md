> 前言
>
> 我是一名打算走音视频路线的android开发者。以此系列文章开始，记录我的音视频开发学习之路
>
> ijkplayer系列文章目录：
> [理解ijkplayer（一）：开始](https://www.jianshu.com/writer#/notebooks/40971763/notes/56760993/preview)
>
> [理解ijkplayer（二）项目结构分析](https://www.jianshu.com/p/b5a2584e03f1)
>
> [理解ijkplayer（三）从Java层开始初始化](https://www.jianshu.com/p/0501be9cf4bf)
>
> [理解ijkplayer（四）拉流](https://www.jianshu.com/p/f633da0db4dd)
>
> [理解ijkplayer（五）解码、播放](https://www.jianshu.com/p/1e10507f18b6)

---



Ijkplayer源码：

https://github.com/bilibili/ijkplayer



学习ijkplayer需要掌握的技能：

1. 简单的c语言基础
2. android开发基础和java jni基础
3. linux或mac开发环境（windows环境下的cygwin连第一个ijkplayer的脚本都无法执行。）



### 1. 开始

按照ijkplayer的readme，将ijkplayer完整的编译并构建so库，理论上是没有问题的。但是我实际执行上还是遇到了一些问题，记录如下：

1. sdk和ndk的环境变量配置：

   可参考[mac开发环境配置](https://www.jianshu.com/p/c3561bb27f43)中的第5点

2. 克隆ffmpeg太慢

   在执行第一个脚本`./init-android.sh`的时候，会去克隆bilibili团队改造的 ffmpeg项目，但是因为某种已知的原因，克隆非常缓慢，解决方式是：将bilibili的ffmpeg项目导入到国内的git仓库，例如码云，然后修改脚本上的url地址指向码云的仓库，就能克隆下来了。

   例如：

   ![](https://upload-images.jianshu.io/upload_images/7177220-cbd9d33614ef4318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   如何将github仓库导入到码云上网一搜就知道了。

3. 此时可以按照官网的教程执行`./init-android.sh`。执行完后，ffmpeg就拉下来了。



### 2. 编译各个架构的so库

```shell
cd android/contrib
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all

cd ..
./compile-ijk.sh all
```

按照上述的命令是编译所有架构的ffmpeg库并以之编译Ijkplayer的所有的so库。

那么如果要编译特定的架构，例如只要armv7a和arm64架构，则：

``` shell
./compile-ffmpeg.sh h
Usage:
  compile-ffmpeg.sh armv5|armv7a|arm64|x86|x86_64
  compile-ffmpeg.sh all|all32
  compile-ffmpeg.sh all64
  compile-ffmpeg.sh clean
  compile-ffmpeg.sh check
```

(什么输入h会出现帮助信息？答案在`compile-ffmpeg.sh`脚本里面)

那么依次输入`./compile-ffmpeg.sh armv7a`编译..，等完成后再输入`./compile-ffmpeg.sh armv64`编译..。

后面再用相同的方式运行`./compile-ijk.sh`脚本即可。



### 3. 开始阅读源码

工具：

1. Visual studio code
2. 删掉编译ffmpeg带来的`build/`目录下对应架构的源码，否则在全局查找某个函数的时候，会显示多个搜索结果，看代码的时候就不方便了。
