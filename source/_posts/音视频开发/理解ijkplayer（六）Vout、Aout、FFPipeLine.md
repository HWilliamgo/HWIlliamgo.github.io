``` c
// ijkmedia/ijkplayer/ff_ffplay_def.h
typedef struct FFPlayer {
		//...
    SDL_Aout *aout;
    SDL_Vout *vout;
    struct IJKFF_Pipeline *pipeline;
    struct IJKFF_Pipenode *node_vdec;
  	//...
}FFPlayer;
```



这篇文章的内容：

分析`FFPlayer`结构体中的`SDL_Aout`、`SDL_Vout`、`IJKFF_Pipeline`和`IJKFF_Pipenode`的作用

此外，这四个结构体，我认为是为了实现多态，详见IBM教程：[技巧：用 C 语言实现程序的多态性](https://www.ibm.com/developerworks/cn/linux/l-cn-cpolym/index.html)

而这里面还有用到`SDL_Vout_Opaque`这类的opaque单词的结构体和指针，这是为了实现封装和对外隐藏细节。

不得不说，ijkplayer用c语言也很好地体现了面向对象的思想。



### SDL_Vout

#### 1. 结构体

##### 1.1 SDL_Vout

``` c
//	ijkmedia/ijksdl/ijksdl_vout.h
struct SDL_Vout {
    SDL_mutex *mutex;

    SDL_Class       *opaque_class;
    SDL_Vout_Opaque *opaque;
    //创建图层，即SDL_VoutOverlay
    SDL_VoutOverlay *(*create_overlay)(int width, int height, int frame_format, SDL_Vout *vout);
    //释放
    void (*free_l)(SDL_Vout *vout);
    //展示图层
    int (*display_overlay)(SDL_Vout *vout, SDL_VoutOverlay *overlay);
    //图层格式
    Uint32 overlay_format;
};
```



##### 1.2 SDL_Vout_Opaque

opaque类似于Java中的内部类，用来向调用者屏蔽该类的内部逻辑



看下这个`SDL_Vout_Opaque`的定义：

用`typedef`定义了一个抽象，那么他的实现在哪里？

``` c
//	ijkmedia/ijksdl/ijksdl_vout.h

//这里引用的是ffmepg后缀的这个头文件，因此用的是软件的方式去创建的SDL_VoutOverlay_Opaque
#include "ffmpeg/ijksdl_inc_ffmpeg.h"

//...
typedef struct SDL_Vout_Opaque SDL_Vout_Opaque;
//...
```



他的实现在两处定义了，分别在硬解和软解的时候使用

``` c
//	硬解
//	ijkmedia/ijksdl/android/ijksdl_vout_overlay_android_mediacodec.c
typedef struct SDL_VoutOverlay_Opaque {
    SDL_mutex *mutex;

    SDL_Vout                   *vout;
    SDL_AMediaCodec            *acodec;

    SDL_AMediaCodecBufferProxy *buffer_proxy;

    Uint16 pitches[AV_NUM_DATA_POINTERS];
    Uint8 *pixels[AV_NUM_DATA_POINTERS];
} SDL_VoutOverlay_Opaque;
```

``` c
// 软解
//	ijkmedia/ijksdl/ffmpeg/ijksdl_vout_overlay_ffmpeg.c
struct SDL_VoutOverlay_Opaque {
    SDL_mutex *mutex;

    AVFrame *managed_frame;
    AVBufferRef *frame_buffer;
    int planes;

    AVFrame *linked_frame;

    Uint16 pitches[AV_NUM_DATA_POINTERS];
    Uint8 *pixels[AV_NUM_DATA_POINTERS];

    int no_neon_warned;

    struct SwsContext *img_convert_ctx;
    int sws_flags;
};
```

而实际上硬解的那个是不会使用的，为什么？因为定义`SDL_VoutOverlay_Opaque`引用的头文件是`ffmpeg/ijksdl_inc_ffmpeg.h`



##### 1.3 SDL_Class

暂时也不清楚这个是做什么的，只保存了一个字符串而已。

``` c
typedef struct SDL_Class {
    const char *name;
} SDL_Class;
```



##### 1.4 SDL_VoutOverlay

``` c
struct SDL_VoutOverlay {
    int w; /**< Read-only */
    int h; /**< Read-only */
    Uint32 format; /**< Read-only */
    int planes; /**< Read-only */
    Uint16 *pitches; /**< in bytes, Read-only */
    Uint8 **pixels; /**< Read-write */

    int is_private;

    int sar_num;
    int sar_den;

    SDL_Class               *opaque_class;
    SDL_VoutOverlay_Opaque  *opaque;

    void    (*free_l)(SDL_VoutOverlay *overlay);
    int     (*lock)(SDL_VoutOverlay *overlay);
    int     (*unlock)(SDL_VoutOverlay *overlay);
    void    (*unref)(SDL_VoutOverlay *overlay);

    int     (*func_fill_frame)(SDL_VoutOverlay *overlay, const AVFrame *frame);
};
```



##### 1.5 SDL_VoutOverlay_Opaque

``` c
#include "ffmpeg/ijksdl_inc_ffmpeg.h"

typedef struct SDL_VoutOverlay_Opaque SDL_VoutOverlay_Opaque;
```

软解：

``` c
//	ijkmedia/ijksdl/ffmpeg/ijksdl_vout_overlay_ffmpeg.c
struct SDL_VoutOverlay_Opaque {
    SDL_mutex *mutex;

    AVFrame *managed_frame;
    AVBufferRef *frame_buffer;
    int planes;

    AVFrame *linked_frame;

    Uint16 pitches[AV_NUM_DATA_POINTERS];
    Uint8 *pixels[AV_NUM_DATA_POINTERS];

    int no_neon_warned;

    struct SwsContext *img_convert_ctx;
    int sws_flags;
};
```

硬解：

``` c
typedef struct SDL_VoutOverlay_Opaque {
    SDL_mutex *mutex;

    SDL_Vout                   *vout;
    SDL_AMediaCodec            *acodec;

    SDL_AMediaCodecBufferProxy *buffer_proxy;

    Uint16 pitches[AV_NUM_DATA_POINTERS];
    Uint8 *pixels[AV_NUM_DATA_POINTERS];
} SDL_VoutOverlay_Opaque;
```

同样的，这里是用的软解。



#### 2. 初始化

#### 3. 使用

``` c
IjkMediaPlayer *ijkmp_android_create(int(*msg_loop)(void*))
{
    //创建IjkMediaPlayer
    IjkMediaPlayer *mp = ijkmp_create(msg_loop);
    if (!mp)
        goto fail;
    //创建视频输出设备，会根据根据硬解还是软件，硬解用MediaCodec创建，软解用FFmpeg创建
mp->ffplayer->vout = SDL_VoutAndroid_CreateForAndroidSurface();
    if (!mp->ffplayer->vout)
        goto fail;
    //创建管道
    mp->ffplayer->pipeline = ffpipeline_create_from_android(mp->ffplayer);
    if (!mp->ffplayer->pipeline)
        goto fail;
    //将创建的视频输出设备vout，赋值到ffplayer->pipeline中
    ffpipeline_set_vout(mp->ffplayer->pipeline, mp->ffplayer->vout);

    return mp;

fail:
    ijkmp_dec_ref_p(&mp);
    return NULL;
}
```

`mp->ffplayer->vout = SDL_VoutAndroid_CreateForAndroidSurface();`

``` c
SDL_Vout *SDL_VoutAndroid_CreateForAndroidSurface()
{
    return SDL_VoutAndroid_CreateForANativeWindow();
}
```

``` c
SDL_Vout *SDL_VoutAndroid_CreateForANativeWindow()
{
    //创建SDL_Vout
    SDL_Vout *vout = SDL_Vout_CreateInternal(sizeof(SDL_Vout_Opaque));
    if (!vout)
        return NULL;

    SDL_Vout_Opaque *opaque = vout->opaque;
    opaque->native_window = NULL;
    if (ISDL_Array__init(&opaque->overlay_manager, 32))
        goto fail;
    if (ISDL_Array__init(&opaque->overlay_pool, 32))
        goto fail;
    //创建egl
    opaque->egl = IJK_EGL_create();
    if (!opaque->egl)
        goto fail;
    //为vout的函数赋值
    vout->opaque_class    = &g_nativewindow_class;
    vout->create_overlay  = func_create_overlay;
    vout->free_l          = func_free_l;
    vout->display_overlay = func_display_overlay;

    return vout;
fail:
    func_free_l(vout);
    return NULL;
}
```

``` c
inline static SDL_Vout *SDL_Vout_CreateInternal(size_t opaque_size)
{
    //分配Vout内存
    SDL_Vout *vout = (SDL_Vout*) calloc(1, sizeof(SDL_Vout));
    if (!vout)
        return NULL;
    //分配Opaue内存
    vout->opaque = calloc(1, opaque_size);
    if (!vout->opaque) {
        free(vout);
        return NULL;
    }
    //创建互斥锁
    vout->mutex = SDL_CreateMutex();
    if (vout->mutex == NULL) {
        free(vout->opaque);
        free(vout);
        return NULL;
    }

    return vout;
}
```



接下来一一看下vout的3个函数在这里被赋值的函数

``` c
vout->create_overlay  = func_create_overlay;
vout->free_l          = func_free_l;
vout->display_overlay = func_display_overlay;
```

``` c
static SDL_VoutOverlay *func_create_overlay(int width, int height, int frame_format, SDL_Vout *vout)
{
    SDL_LockMutex(vout->mutex);
  	//创建SDL_VoutOverlay
    SDL_VoutOverlay *overlay = func_create_overlay_l(width, height, frame_format, vout);
    SDL_UnlockMutex(vout->mutex);
    return overlay;
}
```

``` c
static SDL_VoutOverlay *func_create_overlay_l(int width, int height, int frame_format, SDL_Vout *vout)
{
    switch (frame_format) {
    case IJK_AV_PIX_FMT__ANDROID_MEDIACODEC:
        //如果帧的格式是IJK_AV_PIX_FMT__ANDROID_MEDIACODEC，就用硬解创建
        return SDL_VoutAMediaCodec_CreateOverlay(width, height, vout);
    default:
				//否则用软解创建
        return SDL_VoutFFmpeg_CreateOverlay(width, height, frame_format, vout);
    }
}
```



### SDL_Aout

#### 1. 结构体

``` c
typedef struct SDL_Aout SDL_Aout;
struct SDL_Aout {
    SDL_mutex *mutex;
    double     minimal_latency_seconds;

    SDL_Class       *opaque_class;
    SDL_Aout_Opaque *opaque;
    void (*free_l)(SDL_Aout *vout);
    int (*open_audio)(SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained);
    void (*pause_audio)(SDL_Aout *aout, int pause_on);
    void (*flush_audio)(SDL_Aout *aout);
    void (*set_volume)(SDL_Aout *aout, float left, float right);
    void (*close_audio)(SDL_Aout *aout);

    double (*func_get_latency_seconds)(SDL_Aout *aout);
    void   (*func_set_default_latency_seconds)(SDL_Aout *aout, double latency);

    // optional
    void   (*func_set_playback_rate)(SDL_Aout *aout, float playbackRate);
    void   (*func_set_playback_volume)(SDL_Aout *aout, float playbackVolume);
    int    (*func_get_audio_persecond_callbacks)(SDL_Aout *aout);

    // Android only
    int    (*func_get_audio_session_id)(SDL_Aout *aout);
};
```



#### 2. 初始化

在`ffp_preapare_async_l()函数中`

``` c
int ffp_prepare_async_l(FFPlayer *ffp, const char *file_name)
{
    assert(ffp);
    assert(!ffp->is);
    assert(file_name);
    //针对rtmp和rtsp协议，移除选项”timeout“
    if (av_stristart(file_name, "rtmp", NULL) ||
        av_stristart(file_name, "rtsp", NULL)) {
        // There is total different meaning for 'timeout' option in rtmp
        av_log(ffp, AV_LOG_WARNING, "remove 'timeout' option for rtmp.\n");
        av_dict_set(&ffp->format_opts, "timeout", NULL, 0);
    }

    /* there is a length limit in avformat */
    if (strlen(file_name) + 1 > 1024) {
        av_log(ffp, AV_LOG_ERROR, "%s too long url\n", __func__);
        if (avio_find_protocol_name("ijklongurl:")) {
            av_dict_set(&ffp->format_opts, "ijklongurl-url", file_name, 0);
            file_name = "ijklongurl:";
        }
    }
    //打印版本信息
    av_log(NULL, AV_LOG_INFO, "===== versions =====\n");
    ffp_show_version_str(ffp, "ijkplayer",      ijk_version_info());
    ffp_show_version_str(ffp, "FFmpeg",         av_version_info());
    ffp_show_version_int(ffp, "libavutil",      avutil_version());
    ffp_show_version_int(ffp, "libavcodec",     avcodec_version());
    ffp_show_version_int(ffp, "libavformat",    avformat_version());
    ffp_show_version_int(ffp, "libswscale",     swscale_version());
    ffp_show_version_int(ffp, "libswresample",  swresample_version());
    av_log(NULL, AV_LOG_INFO, "===== options =====\n");
    ffp_show_dict(ffp, "player-opts", ffp->player_opts);
    ffp_show_dict(ffp, "format-opts", ffp->format_opts);
    ffp_show_dict(ffp, "codec-opts ", ffp->codec_opts);
    ffp_show_dict(ffp, "sws-opts   ", ffp->sws_dict);
    ffp_show_dict(ffp, "swr-opts   ", ffp->swr_opts);
    av_log(NULL, AV_LOG_INFO, "===================\n");
    //设置播放器选项
    av_opt_set_dict(ffp, &ffp->player_opts);
    //如果ffplayer->aout==null，那么久打开音频输出设备。前面的初始化代码是没有为这个赋值过的，所以第一次调用肯定会返回true.
    if (!ffp->aout) {
        ffp->aout = ffpipeline_open_audio_output(ffp->pipeline, ffp);
        if (!ffp->aout)
            return -1;
    }

#if CONFIG_AVFILTER
    if (ffp->vfilter0) {
        GROW_ARRAY(ffp->vfilters_list, ffp->nb_vfilters);
        ffp->vfilters_list[ffp->nb_vfilters - 1] = ffp->vfilter0;
    }
#endif

    VideoState *is = stream_open(ffp, file_name, NULL);
    if (!is) {
        av_log(NULL, AV_LOG_WARNING, "ffp_prepare_async_l: stream_open failed OOM");
        return EIJK_OUT_OF_MEMORY;
    }

    ffp->is = is;
    ffp->input_filename = av_strdup(file_name);
    return 0;
}
```

即这句：

``` c
//如果ffplayer->aout==null，那么久打开音频输出设备。前面的初始化代码是没有为这个赋值过的，所以第一次调用肯定会返回true.
    if (!ffp->aout) {
        ffp->aout = ffpipeline_open_audio_output(ffp->pipeline, ffp);
        if (!ffp->aout)
            return -1;
    }
```

``` c
SDL_Aout *ffpipeline_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    //借助pipeline的方法
    return pipeline->func_open_audio_output(pipeline, ffp);
}
```

这个地方要使用`IJKFF_Pipeline`的方法，而``IJKFF_Pipeline``是在创建播放器的时候创建的。

``` c
static SDL_Aout *func_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    SDL_Aout *aout = NULL;
    if (ffp->opensles) {
        aout = SDL_AoutAndroid_CreateForOpenSLES();
    } else {
      	//一般不会用opensles，都是默认用的android的AudioTrack来创建Aout
        aout = SDL_AoutAndroid_CreateForAudioTrack();
    }
    if (aout)
        SDL_AoutSetStereoVolume(aout, pipeline->opaque->left_volume, pipeline->opaque->right_volume);
    return aout;
}
```

``` c
SDL_Aout *SDL_AoutAndroid_CreateForAudioTrack()
{
    SDL_Aout *aout = SDL_Aout_CreateInternal(sizeof(SDL_Aout_Opaque));
    if (!aout)
        return NULL;

    SDL_Aout_Opaque *opaque = aout->opaque;
    opaque->wakeup_cond  = SDL_CreateCond();
    opaque->wakeup_mutex = SDL_CreateMutex();
    opaque->speed        = 1.0f;

    aout->opaque_class = &g_audiotrack_class;
    aout->free_l       = aout_free_l;
    aout->open_audio   = aout_open_audio;
    aout->pause_audio  = aout_pause_audio;
    aout->flush_audio  = aout_flush_audio;
    aout->set_volume   = aout_set_volume;
    aout->close_audio  = aout_close_audio;
    aout->func_get_audio_session_id = aout_get_audio_session_id;
    aout->func_set_playback_rate    = func_set_playback_rate;

    return aout;
}
```

那么这里看一下这个`aoujt->open_audio`函数：

``` c
static int aout_open_audio(SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained)
{
    // SDL_Aout_Opaque *opaque = aout->opaque;
    JNIEnv *env = NULL;
    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("aout_open_audio: AttachCurrentThread: failed");
        return -1;
    }

    return aout_open_audio_n(env, aout, desired, obtained);
}
```

``` c
static int aout_open_audio_n(JNIEnv *env, SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained)
{
    assert(desired);
    SDL_Aout_Opaque *opaque = aout->opaque;

    opaque->spec = *desired;
    opaque->atrack = SDL_Android_AudioTrack_new_from_sdl_spec(env, desired);
    if (!opaque->atrack) {
        ALOGE("aout_open_audio_n: failed to new AudioTrcak()");
        return -1;
    }

    opaque->buffer_size = SDL_Android_AudioTrack_get_min_buffer_size(opaque->atrack);
    if (opaque->buffer_size <= 0) {
        ALOGE("aout_open_audio_n: failed to getMinBufferSize()");
        SDL_Android_AudioTrack_free(env, opaque->atrack);
        opaque->atrack = NULL;
        return -1;
    }

    opaque->buffer = malloc(opaque->buffer_size);
    if (!opaque->buffer) {
        ALOGE("aout_open_audio_n: failed to allocate buffer");
        SDL_Android_AudioTrack_free(env, opaque->atrack);
        opaque->atrack = NULL;
        return -1;
    }

    if (obtained) {
        SDL_Android_AudioTrack_get_target_spec(opaque->atrack, obtained);
        SDLTRACE("audio target format fmt:0x%x, channel:0x%x", (int)obtained->format, (int)obtained->channels);
    }

    opaque->audio_session_id = SDL_Android_AudioTrack_getAudioSessionId(env, opaque->atrack);
    ALOGI("audio_session_id = %d\n", opaque->audio_session_id);

    opaque->pause_on = 1;
    opaque->abort_request = 0;
    //创建音频输出线程
    opaque->audio_tid = SDL_CreateThreadEx(&opaque->_audio_tid, aout_thread, aout, "ff_aout_android");
    if (!opaque->audio_tid) {
        ALOGE("aout_open_audio_n: failed to create audio thread");
        SDL_Android_AudioTrack_free(env, opaque->atrack);
        opaque->atrack = NULL;
        return -1;
    }

    return 0;
}
```

那么这里看下这个音频输出线程做了什么：

``` c
static int aout_thread(void *arg)
{
    SDL_Aout *aout = arg;
    // SDL_Aout_Opaque *opaque = aout->opaque;
    JNIEnv *env = NULL;

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("aout_thread: SDL_AndroidJni_SetupEnv: failed");
        return -1;
    }

    return aout_thread_n(env, aout);
}
```

``` c
static int aout_thread_n(JNIEnv *env, SDL_Aout *aout)
{
    SDL_Aout_Opaque *opaque = aout->opaque;
    SDL_Android_AudioTrack *atrack = opaque->atrack;
    SDL_AudioCallback audio_cblk = opaque->spec.callback;
    void *userdata = opaque->spec.userdata;
    uint8_t *buffer = opaque->buffer;
    int copy_size = 256;

    assert(atrack);
    assert(buffer);

    SDL_SetThreadPriority(SDL_THREAD_PRIORITY_HIGH);

    if (!opaque->abort_request && !opaque->pause_on)
        SDL_Android_AudioTrack_play(env, atrack);
    //只要没有中断请求，就无限循环
    while (!opaque->abort_request) {
        SDL_LockMutex(opaque->wakeup_mutex);
        if (!opaque->abort_request && opaque->pause_on) {
            //暂停
            SDL_Android_AudioTrack_pause(env, atrack);
            while (!opaque->abort_request && opaque->pause_on) {
                SDL_CondWaitTimeout(opaque->wakeup_cond, opaque->wakeup_mutex, 1000);
            }
            if (!opaque->abort_request && !opaque->pause_on) {
                if (opaque->need_flush) {
                    opaque->need_flush = 0;
                    //flush
                    SDL_Android_AudioTrack_flush(env, atrack);
                }
                //播放
                SDL_Android_AudioTrack_play(env, atrack);
            }
        }
        if (opaque->need_flush) {
            opaque->need_flush = 0;
            SDL_Android_AudioTrack_flush(env, atrack);
        }
        if (opaque->need_set_volume) {
            opaque->need_set_volume = 0;
            SDL_Android_AudioTrack_set_volume(env, atrack, opaque->left_volume, opaque->right_volume);
        }
        if (opaque->speed_changed) {
            opaque->speed_changed = 0;
            SDL_Android_AudioTrack_setSpeed(env, atrack, opaque->speed);
        }
        SDL_UnlockMutex(opaque->wakeup_mutex);

        audio_cblk(userdata, buffer, copy_size);
        if (opaque->need_flush) {
            SDL_Android_AudioTrack_flush(env, atrack);
            opaque->need_flush = false;
        }

        if (opaque->need_flush) {
            opaque->need_flush = 0;
            SDL_Android_AudioTrack_flush(env, atrack);
        } else {
            int written = SDL_Android_AudioTrack_write(env, atrack, buffer, copy_size);
            if (written != copy_size) {
                ALOGW("AudioTrack: not all data copied %d/%d", (int)written, (int)copy_size);
            }
        }

        // TODO: 1 if callback return -1 or 0
    }

    SDL_Android_AudioTrack_free(env, atrack);
    return 0;
}
```

这里看下这个播放是在干嘛：

``` c
void SDL_Android_AudioTrack_play(JNIEnv *env, SDL_Android_AudioTrack *atrack)
{
    SDLTRACE("%s", __func__);
    J4AC_AudioTrack__play__catchAll(env, atrack->thiz);
}
```

``` c
// ijkmedia/ijkj4a/j4a/class/android/media/AudioTrack.h
#define J4AC_AudioTrack__play__catchAll J4AC_android_media_AudioTrack__play__catchAll
```

``` c
void J4AC_android_media_AudioTrack__play__catchAll(JNIEnv *env, jobject thiz)
{
    J4AC_android_media_AudioTrack__play(env, thiz);
    J4A_ExceptionCheck__catchAll(env);
}
```

``` c
void J4AC_android_media_AudioTrack__play(JNIEnv *env, jobject thiz)
{	
  	//调用了jni方法，即通过c去调用java里的方法了。
    (*env)->CallVoidMethod(env, thiz, class_J4AC_android_media_AudioTrack.method_play);
}
```

而这个`class_J4AC_android_media_AudioTrack.method_play`是：

``` c
//	ijkmedia/ijkj4a/j4a/class/android/media/AudioTrack.c
int J4A_loadClass__J4AC_android_media_AudioTrack(JNIEnv *env)
{
    //...
    class_id = class_J4AC_android_media_AudioTrack.id;
    name     = "play";
    sign     = "()V";
    class_J4AC_android_media_AudioTrack.method_play = J4A_GetMethodID__catchAll(env, class_id, name, sign);
    //...
}
```

那么的确是调用的java层的`AudioTrack#play()`



#### 3. 使用

``` c
switch (avctx->codec_type) {
  case AVMEDIA_TYPE_AUDIO:
    	//audio_open里面会去调用到AudioTrack.java # play()
      if ((ret = audio_open(ffp, channel_layout, nb_channels, sample_rate, &is->audio_tgt)) < 0)
        //decoder初始化
      decoder_init(&is->auddec, avctx, &is->audioq, is->continue_read_thread);
      //decoder启动，启动audio_thread线程
      if ((ret = decoder_start(&is->auddec, audio_thread, ffp, "ff_audio_dec")) < 0)
}
```





### IJKFF_Pipeline

#### 1. 结构体

##### 1.1 IJKFF_Pipeline

``` c
typedef struct IJKFF_Pipeline IJKFF_Pipeline;
struct IJKFF_Pipeline {
    SDL_Class             *opaque_class;
    IJKFF_Pipeline_Opaque *opaque;
		//销毁
    void            (*func_destroy)             (IJKFF_Pipeline *pipeline);
  	//打开视频解码器
    IJKFF_Pipenode *(*func_open_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
  	//打开音频解码器
    SDL_Aout       *(*func_open_audio_output)   (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
  	//初始化视频解码器
    IJKFF_Pipenode *(*func_init_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
  	//配置视频解码器
    int           (*func_config_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
};
```

##### 1.2 IJKFF_Pipeline_Opaque

``` c
typedef struct IJKFF_Pipeline_Opaque {
    FFPlayer      *ffp;
    SDL_mutex     *surface_mutex;
    jobject        jsurface;
    volatile bool  is_surface_need_reconfigure;

    bool         (*mediacodec_select_callback)(void *opaque, ijkmp_mediacodecinfo_context *mcc);
    void          *mediacodec_select_callback_opaque;

    SDL_Vout      *weak_vout;

    float          left_volume;
    float          right_volume;
} IJKFF_Pipeline_Opaque;
```





#### 2. 初始化

``` c
IjkMediaPlayer *ijkmp_android_create(int(*msg_loop)(void*))
{
		//...
    mp->ffplayer->pipeline = ffpipeline_create_from_android(mp->ffplayer);
    //...
}
```



``` c
IJKFF_Pipeline *ffpipeline_create_from_android(FFPlayer *ffp)
{
    ALOGD("ffpipeline_create_from_android()\n");
    //分配内存
    IJKFF_Pipeline *pipeline = ffpipeline_alloc(&g_pipeline_class, sizeof(IJKFF_Pipeline_Opaque));
    if (!pipeline)
        return pipeline;
    //初始化opaque
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    opaque->ffp                   = ffp;
    opaque->surface_mutex         = SDL_CreateMutex();
    opaque->left_volume           = 1.0f;
    opaque->right_volume          = 1.0f;
    if (!opaque->surface_mutex) {
        ALOGE("ffpipeline-android:create SDL_CreateMutex failed\n");
        goto fail;
    }
    //初始化pipeline中的每个函数
    pipeline->func_destroy              = func_destroy;
    pipeline->func_open_video_decoder   = func_open_video_decoder;
    pipeline->func_open_audio_output    = func_open_audio_output;
    pipeline->func_init_video_decoder   = func_init_video_decoder;
    pipeline->func_config_video_decoder = func_config_video_decoder;

    return pipeline;
fail:
    ffpipeline_free_p(&pipeline);
    return NULL;
}
```



一个一个来看一下pipeline中的函数的作用：

##### 2.1 func_destroy()

``` c
static void func_destroy(IJKFF_Pipeline *pipeline)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    JNIEnv *env = NULL;

    SDL_DestroyMutexP(&opaque->surface_mutex);

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("amediacodec-pipeline:destroy: SetupThreadEnv failed\n");
        goto fail;
    }
    //变量并释放IJKFF_Pipeline_Opaque.jsurface
    SDL_JNI_DeleteGlobalRefP(env, &opaque->jsurface);
fail:
    return;
}
```

``` c
//用env指针变量，删除obj_ptr的jni全局引用
void SDL_JNI_DeleteGlobalRefP(JNIEnv *env, jobject *obj_ptr)
{
    if (!obj_ptr || !*obj_ptr)
        return;
		//jni方法，删除全局引用
    (*env)->DeleteGlobalRef(env, *obj_ptr);
    *obj_ptr = NULL;
}
```



##### 2.2 func_open_video_decoder()

``` c
static IJKFF_Pipenode *func_open_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    IJKFF_Pipenode        *node = NULL;

    if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2)
        //从硬解中创建解码器
        node = ffpipenode_create_video_decoder_from_android_mediacodec(ffp, pipeline, opaque->weak_vout);
    if (!node) {
        //从ffplay中创建解码器，即ffmpeg的解码器
        node = ffpipenode_create_video_decoder_from_ffplay(ffp);
    }

    return node;
}
```

这里很有意思，创建解码器，而返回的对象是`IJKFF_Pipenode`，这是否说明``IJKFF_Pipenode``就是一个解码器的抽象？



##### 2.3 func_open_audio_output

``` c
static SDL_Aout *func_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    SDL_Aout *aout = NULL;
    if (ffp->opensles) {
        aout = SDL_AoutAndroid_CreateForOpenSLES();
    } else {
      	//一般不会用opensles，都是默认用的android的AudioTrack来创建Aout
        aout = SDL_AoutAndroid_CreateForAudioTrack();
    }
    if (aout)
        SDL_AoutSetStereoVolume(aout, pipeline->opaque->left_volume, pipeline->opaque->right_volume);
    return aout;
}
```



``` c
SDL_Aout *SDL_AoutAndroid_CreateForAudioTrack()
{
  	//在这里创建并分配了SDL_Aout结构体
    SDL_Aout *aout = SDL_Aout_CreateInternal(sizeof(SDL_Aout_Opaque));
    if (!aout)
        return NULL;

    SDL_Aout_Opaque *opaque = aout->opaque;
    opaque->wakeup_cond  = SDL_CreateCond();
    opaque->wakeup_mutex = SDL_CreateMutex();
    opaque->speed        = 1.0f;

    aout->opaque_class = &g_audiotrack_class;
    aout->free_l       = aout_free_l;
    aout->open_audio   = aout_open_audio;
    aout->pause_audio  = aout_pause_audio;
    aout->flush_audio  = aout_flush_audio;
    aout->set_volume   = aout_set_volume;
    aout->close_audio  = aout_close_audio;
    aout->func_get_audio_session_id = aout_get_audio_session_id;
    aout->func_set_playback_rate    = func_set_playback_rate;

    return aout;
}
```



##### 2.4 func_init_video_decoder

``` c
static IJKFF_Pipenode *func_init_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    IJKFF_Pipenode        *node = NULL;

    if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2)
      	//如果是硬解，则要再初始化一下，如果是ffmpeg的软解，就不需要了。
        node = ffpipenode_init_decoder_from_android_mediacodec(ffp, pipeline, opaque->weak_vout);

    return node;
}
```



##### 2.5 func_config_video_decoder

``` c
static int func_config_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    int                       ret = NULL;

    if (ffp->node_vdec) {
        ret = ffpipenode_config_from_android_mediacodec(ffp, pipeline, opaque->weak_vout, ffp->node_vdec);
    }

    return ret;
}
```





#### 3. 使用

视频解码线程开始之前，用`ffpipeline_open_video_decoder`创建一个解码器。

``` c
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
  //...
  case AVMEDIA_TYPE_VIDEO:
  	//decoder初始化
		decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
    ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
	  //解码器开始
    if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
    	goto out;
  //...
}
```



### IJKFF_Pipenode

这个称为管道节点结构体包含的`func_run_sync()`函数是用来运行解码线程的。因此该结构体对底层ffmpeg来说，也是底层ffmpeg的一层抽象了。

#### 1. 结构体

``` c
typedef struct IJKFF_Pipenode IJKFF_Pipenode;
struct IJKFF_Pipenode {
    SDL_mutex *mutex;
    void *opaque;

    void (*func_destroy) (IJKFF_Pipenode *node);
    int  (*func_run_sync)(IJKFF_Pipenode *node);
    int  (*func_flush)   (IJKFF_Pipenode *node); // optional
};
```



#### 2. 初始化

``` c
IJKFF_Pipenode *ffpipenode_create_video_decoder_from_ffplay(FFPlayer *ffp)
{
    //分配IJKFF_Pipenode的内存
    IJKFF_Pipenode *node = ffpipenode_alloc(sizeof(IJKFF_Pipenode_Opaque));
    if (!node)
        return node;

    IJKFF_Pipenode_Opaque *opaque = node->opaque;
    opaque->ffp         = ffp;
		//为node的函数赋值
    node->func_destroy  = func_destroy;
    node->func_run_sync = func_run_sync;
  
    ffp_set_video_codec_info(ffp, AVCODEC_MODULE_NAME, avcodec_get_name(ffp->is->viddec.avctx->codec_id));
    ffp->stat.vdec_type = FFP_PROPV_DECODER_AVCODEC;
    return node;
}
```



``` c
static void func_destroy(IJKFF_Pipenode *node)
{
    // do nothing
}
static int func_run_sync(IJKFF_Pipenode *node)
{
    IJKFF_Pipenode_Opaque *opaque = node->opaque;

    return ffp_video_thread(opaque->ffp);
}
```

``` c
int ffp_video_thread(FFPlayer *ffp)
{
    return ffplay_video_thread(ffp);
}
```



``` c
static int ffplay_video_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFrame *frame = av_frame_alloc();
    double pts;
    double duration;
    int ret;
    AVRational tb = is->video_st->time_base;
    AVRational frame_rate = av_guess_frame_rate(is->ic, is->video_st, NULL);
    int64_t dst_pts = -1;
    int64_t last_dst_pts = -1;
    int retry_convert_image = 0;
    int convert_frame_count = 0;

    ffp_notify_msg2(ffp, FFP_MSG_VIDEO_ROTATION_CHANGED, ffp_get_video_rotate_degrees(ffp));

    if (!frame) {
        return AVERROR(ENOMEM);
    }

    for (;;) {
      	//获取解码后的数据AVFrame数据
        ret = get_video_frame(ffp, frame);
        if (ret < 0)
            goto the_end;
        if (!ret)
            continue;

        if (ffp->get_frame_mode) {
            if (!ffp->get_img_info || ffp->get_img_info->count <= 0) {
                av_frame_unref(frame);
                continue;
            }

            last_dst_pts = dst_pts;

            if (dst_pts < 0) {
                dst_pts = ffp->get_img_info->start_time;
            } else {
                dst_pts += (ffp->get_img_info->end_time - ffp->get_img_info->start_time) / (ffp->get_img_info->num - 1);
            }

            pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
            pts = pts * 1000;
            if (pts >= dst_pts) {
                while (retry_convert_image <= MAX_RETRY_CONVERT_IMAGE) {
                    ret = convert_image(ffp, frame, (int64_t)pts, frame->width, frame->height);
                    if (!ret) {
                        convert_frame_count++;
                        break;
                    }
                    retry_convert_image++;
                    av_log(NULL, AV_LOG_ERROR, "convert image error retry_convert_image = %d\n", retry_convert_image);
                }

                retry_convert_image = 0;
                if (ret || ffp->get_img_info->count <= 0) {
                    if (ret) {
                        av_log(NULL, AV_LOG_ERROR, "convert image abort ret = %d\n", ret);
                        ffp_notify_msg3(ffp, FFP_MSG_GET_IMG_STATE, 0, ret);
                    } else {
                        av_log(NULL, AV_LOG_INFO, "convert image complete convert_frame_count = %d\n", convert_frame_count);
                    }
                    goto the_end;
                }
            } else {
                dst_pts = last_dst_pts;
            }
            av_frame_unref(frame);
            continue;
        }
            duration = (frame_rate.num && frame_rate.den ? av_q2d((AVRational){frame_rate.den, frame_rate.num}) : 0);
            pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
      			//将frame数据进入到picture_queue。即渲染队列
            ret = queue_picture(ffp, frame, pts, duration, frame->pkt_pos, is->viddec.pkt_serial);
            av_frame_unref(frame);

        if (ret < 0)
            goto the_end;
    }
 the_end:
    av_log(NULL, AV_LOG_INFO, "convert image convert_frame_count = %d\n", convert_frame_count);
    av_frame_free(&frame);
    return 0;
}
```



#### 3. 使用

主要就是他的`func_run_sync()`方法被用来解码视频帧并入队渲染队列。这个操作发生在：

``` c
static int video_thread(void *arg)
{
    FFPlayer *ffp = (FFPlayer *)arg;
    int       ret = 0;
		//如果node_vdec不为null。
    if (ffp->node_vdec) {
      	//解码操作
        ret = ffpipenode_run_sync(ffp->node_vdec);
    }
    return ret;
}
```

 那么我们来看一下解码的时候他的逻辑：

``` c
//	ijkmedia/ijkplayer/pipeline/ffpipenode_ffplay_vdec.c
static int func_run_sync(IJKFF_Pipenode *node)
{
    IJKFF_Pipenode_Opaque *opaque = node->opaque;

    return ffp_video_thread(opaque->ffp);
}
```

``` c
//	ijkmedia/ijkplayer/ff_ffplay.c
int ffp_video_thread(FFPlayer *ffp)
{
    return ffplay_video_thread(ffp);
}
```


