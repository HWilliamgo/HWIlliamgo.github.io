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



### 1. 整体结构

``` shell
.
├── android/   #android项目的demo和so库的输出路径
├── config/
├── doc/
├── extra/		#ijkplayer依赖的第三方库的源码,例如ffmpeg
├── ijkmedia/ #ijkplayer源码
├── ijkprof/
├── ios/
├── tools/
├── COPYING.GPLv2
├── COPYING.GPLv3
├── COPYING.LGPLv2.1
├── COPYING.LGPLv2.1.txt -> COPYING.LGPLv2.1
├── COPYING.LGPLv3
├── MODULE_LICENSE_APACHE2
├── NEWS.md
├── NOTICE
├── README.md
├── compile-android-j4a.sh*
├── init-android-exo.sh*
├── init-android-j4a.sh*
├── init-android-libsoxr.sh*
├── init-android-libyuv.sh*
├── init-android-openssl.sh*
├── init-android-prof.sh*
├── init-android-soundtouch.sh*
├── init-android.sh*
├── init-config.sh*
├── init-ios-openssl.sh*
├── init-ios.sh*
└── version.sh*
```

看源码只需要重点关注`/ijkmedia`和`extra/`

### 2. 入口：ijkplayer_jni.c

`ijkmedia/ijkplayer/android/ijkplayer_jni.c`

这是android的入口文件，用动态加载的方式声明了jni方法：

``` c
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved){
  //.
}
```



在`g_methods`数组中动态注册了jni方法。

``` c
static JNINativeMethod g_methods[] = {
    {
        "_setDataSource",
        "(Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/String;)V",
        (void *) IjkMediaPlayer_setDataSourceAndHeaders
    },
    { "_setDataSourceFd",       "(I)V",     (void *) IjkMediaPlayer_setDataSourceFd },
    { "_setDataSource",         "(Ltv/danmaku/ijk/media/player/misc/IMediaDataSource;)V", (void *)IjkMediaPlayer_setDataSourceCallback },
    { "_setAndroidIOCallback",  "(Ltv/danmaku/ijk/media/player/misc/IAndroidIO;)V", (void *)IjkMediaPlayer_setAndroidIOCallback },

    { "_setVideoSurface",       "(Landroid/view/Surface;)V", (void *) IjkMediaPlayer_setVideoSurface },
    { "_prepareAsync",          "()V",      (void *) IjkMediaPlayer_prepareAsync },
    { "_start",                 "()V",      (void *) IjkMediaPlayer_start },
    { "_stop",                  "()V",      (void *) IjkMediaPlayer_stop },
    { "seekTo",                 "(J)V",     (void *) IjkMediaPlayer_seekTo },
    { "_pause",                 "()V",      (void *) IjkMediaPlayer_pause },
    { "isPlaying",              "()Z",      (void *) IjkMediaPlayer_isPlaying },
    { "getCurrentPosition",     "()J",      (void *) IjkMediaPlayer_getCurrentPosition },
    { "getDuration",            "()J",      (void *) IjkMediaPlayer_getDuration },
    { "_release",               "()V",      (void *) IjkMediaPlayer_release },
    { "_reset",                 "()V",      (void *) IjkMediaPlayer_reset },
    { "setVolume",              "(FF)V",    (void *) IjkMediaPlayer_setVolume },
    { "getAudioSessionId",      "()I",      (void *) IjkMediaPlayer_getAudioSessionId },
    { "native_init",            "()V",      (void *) IjkMediaPlayer_native_init },
    { "native_setup",           "(Ljava/lang/Object;)V", (void *) IjkMediaPlayer_native_setup },
    { "native_finalize",        "()V",      (void *) IjkMediaPlayer_native_finalize },

    { "_setOption",             "(ILjava/lang/String;Ljava/lang/String;)V", (void *) IjkMediaPlayer_setOption },
    { "_setOption",             "(ILjava/lang/String;J)V",                  (void *) IjkMediaPlayer_setOptionLong },

    { "_getColorFormatName",    "(I)Ljava/lang/String;",    (void *) IjkMediaPlayer_getColorFormatName },
    { "_getVideoCodecInfo",     "()Ljava/lang/String;",     (void *) IjkMediaPlayer_getVideoCodecInfo },
    { "_getAudioCodecInfo",     "()Ljava/lang/String;",     (void *) IjkMediaPlayer_getAudioCodecInfo },
    { "_getMediaMeta",          "()Landroid/os/Bundle;",    (void *) IjkMediaPlayer_getMediaMeta },
    { "_setLoopCount",          "(I)V",                     (void *) IjkMediaPlayer_setLoopCount },
    { "_getLoopCount",          "()I",                      (void *) IjkMediaPlayer_getLoopCount },
    { "_getPropertyFloat",      "(IF)F",                    (void *) ijkMediaPlayer_getPropertyFloat },
    { "_setPropertyFloat",      "(IF)V",                    (void *) ijkMediaPlayer_setPropertyFloat },
    { "_getPropertyLong",       "(IJ)J",                    (void *) ijkMediaPlayer_getPropertyLong },
    { "_setPropertyLong",       "(IJ)V",                    (void *) ijkMediaPlayer_setPropertyLong },
    { "_setStreamSelected",     "(IZ)V",                    (void *) ijkMediaPlayer_setStreamSelected },

    { "native_profileBegin",    "(Ljava/lang/String;)V",    (void *) IjkMediaPlayer_native_profileBegin },
    { "native_profileEnd",      "()V",                      (void *) IjkMediaPlayer_native_profileEnd },

    { "native_setLogLevel",     "(I)V",                     (void *) IjkMediaPlayer_native_setLogLevel },
    { "_setFrameAtTime",        "(Ljava/lang/String;JJII)V", (void *) IjkMediaPlayer_setFrameAtTime },
};
```



我们可以看到规则，每当java层调用jni方法例如：

| java调用的jni方法 | 对应Ijkplayer内部的方法                |
| ----------------- | -------------------------------------- |
| _setDataSource    | IjkMediaPlayer_setDataSourceAndHeaders |
| _start            | IjkMediaPlayer_start                   |



即Java层调用的`XXX`方法，对应在ijkplayer的c层是`IjkMediaPlayer_XXX`。

而`IjkMediaPlayer_XXX`方法全部生命在`ijkplayer_jni.c`文件中，也难怪后者有1200多行了。



### 3. 中转,调度：ijkplayer.c

ijkmedia/ijkplayer/ijkplayer.c

我们拿一个jni方法来看下：

``` c
//ijkplayer_jni.c

static void
IjkMediaPlayer_start(JNIEnv *env, jobject thiz)
{
    MPTRACE("%s\n", __func__);
  	//获取IjkMediaPlayer对象（是个结构体，但是就是一个对象）
    IjkMediaPlayer *mp = jni_get_media_player(env, thiz);
    JNI_CHECK_GOTO(mp, env, "java/lang/IllegalStateException", "mpjni: start: null mp", LABEL_RETURN);
		//跳转到了ijkplayer.c的函数了
    ijkmp_start(mp);

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```



Ijkplayer_jin.c的每一个Jni映射方法几乎都是这样，拿到一个IjkMediaPlayer对象，然后跳转到ijkplayer.c中的函数。

注意:Ijkplayer_jin.c是通过

``` c
#include "ijkplayer_android.h"
```

而`ijkplayer_android.h`中，又

``` c
#include "../ijkplayer.h"
```

所以，具备了调用ijkpalyer.h的能力的，而`ijkplayer.h`的实现是`ijkplayer.c`因此主要看后者就行了。

而在这一层映射中，函数命名也有对应关系：

| ijkplayer_jin.c             | ijkplayer.h / ijkpalyer.c |
| --------------------------- | ------------------------- |
| IjkMediaPlayer_prepareAsync | ijkmp_prepare_async       |
| IjkMediaPlayer_start        | ijkmp_start               |



### 4. 封装：ff_ffplay.c

ijkplayer.c中的代码会调用到ff_ffplay.c中的代码，以及类似的ff_ffxxx.c中的代码：



即以ff开头的文件中的函数：

![](https://upload-images.jianshu.io/upload_images/7177220-b8c20a665cd7c027.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


