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



由于篇幅的原因，因此这一篇文章是接着上一篇继续写的。

上一篇文章分析完了：

1. JNI_Onload()
2. native_init()
3. native_setup()
4. _setDataSource()
5. _setVideoSurface





### 1  _prepareAsync()

播放器的异步准备。这是初始化阶段中最复杂，最重要的函数。

``` c
//	ijkmedia/ijkplayer/android/ijkplayer_jni.c
static void
IjkMediaPlayer_prepareAsync(JNIEnv *env, jobject thiz)
{
    MPTRACE("%s\n", __func__);
    int retval = 0;
    IjkMediaPlayer *mp = jni_get_media_player(env, thiz);
    JNI_CHECK_GOTO(mp, env, "java/lang/IllegalStateException", "mpjni: prepareAsync: null mp", LABEL_RETURN);

    retval = ijkmp_prepare_async(mp);
    IJK_CHECK_MPRET_GOTO(retval, env, LABEL_RETURN);

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```

``` c
//	ijkmedia/ijkplayer/ijkplayer.c
int ijkmp_prepare_async(IjkMediaPlayer *mp)
{
    assert(mp);
    MPTRACE("ijkmp_prepare_async()\n");
    pthread_mutex_lock(&mp->mutex);
    int retval = ijkmp_prepare_async_l(mp);
    pthread_mutex_unlock(&mp->mutex);
    MPTRACE("ijkmp_prepare_async()=%d\n", retval);
    return retval;
}
```

``` c
static int ijkmp_prepare_async_l(IjkMediaPlayer *mp)
{
    assert(mp);

    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_IDLE);
    // MPST_RET_IF_EQ(mp->mp_state, MP_STATE_INITIALIZED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_ASYNC_PREPARING);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_PREPARED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_STARTED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_PAUSED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_COMPLETED);
    // MPST_RET_IF_EQ(mp->mp_state, MP_STATE_STOPPED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_ERROR);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_END);
    //声明url不为空
    assert(mp->data_source);
    //改变播放器状态到MP_STATE_ASYNC_PREPARING
    ijkmp_change_state_l(mp, MP_STATE_ASYNC_PREPARING);
    //消息队列开始
    msg_queue_start(&mp->ffplayer->msg_queue);

    // released in msg_loop
    ijkmp_inc_ref(mp);
    //创建并启动消息线程，开启循环来读取消息队列的消息。
    mp->msg_thread = SDL_CreateThreadEx(&mp->_msg_thread, ijkmp_msg_loop, mp, "ff_msg_loop");
    // msg_thread is detached inside msg_loop
    // TODO: 9 release weak_thiz if pthread_create() failed;
    //逻辑跳转到ff_ffplay.c
    int retval = ffp_prepare_async_l(mp->ffplayer, mp->data_source);
    if (retval < 0) {
        //出错，则抛出MP_STATE_ERROR
        ijkmp_change_state_l(mp, MP_STATE_ERROR);
        return retval;
    }

    return 0;
}
```

看到创建消息线程的那一句的`ijkmp_msg_loop`函数

``` c
//这句函数会在线程被启动的时候调用，类似于Thread中的Runnable
static int ijkmp_msg_loop(void *arg)
{
    IjkMediaPlayer *mp = arg;
  	//调用mp的msg_loop函数。
    int ret = mp->msg_loop(arg);
    return ret;
}
```

`mp->msg_loop`这个函数，在前面播放器被创建的时候被赋值，在`3.3.2`中。

那么这时开启了消息循环线程，并且prepare的逻辑跳转到了`ff_ffplay.c`中



``` c
//	ijkmedia/ijkplayer/ff_ffplay.c
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
		//打开流，并返回一个VideoState的结构体
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



### 2 打开音频输出设备

```c
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

而`ffp->pipeline`是在创建播放器IjkMediaPlayer的时候，在创建完`ffplayer`，和`ffplyaer->vout`一起创建的，在`3.3.2`有如下的代码：

``` c
//	ijkmedia/ijkplayer/android/ijkplayer_android.c
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
    //暂时不太理解这个叫做”管道“的东西是什么
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

那么我们看到`pipeline->func_open_audio_output(pipeline, ffp);`的这个方法：我一点进去直接跳转到`IJKFF_Pipeline`的结构体的定义来了。

``` c
struct IJKFF_Pipeline {
    SDL_Class             *opaque_class;
    IJKFF_Pipeline_Opaque *opaque;

    void            (*func_destroy)             (IJKFF_Pipeline *pipeline);
    IJKFF_Pipenode *(*func_open_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
  	//我们要看的是这个方法，名叫：打开音频输出设备。
    SDL_Aout       *(*func_open_audio_output)   (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
    IJKFF_Pipenode *(*func_init_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
    int           (*func_config_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
};
```

这个函数应该是在某个地方被赋值了，我们得找一下，全局搜索关键字：`func_open_audio_output`

![](https://upload-images.jianshu.io/upload_images/7177220-39dc9db5869ee765.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

全局搜索中出现了3个对`func_open_audio_output`赋值的语句，分别出现在

1. `ffpipleline_android.c`
2. `ffpipeline_ffplay.c`
3. `ffpipeline_ios.c`

其中貌似android和ios平台有各自的赋值规则，然后又一个中立的赋值的地方，我们先看这个平台无关的中立的赋值的地方：

``` c
static SDL_Aout *func_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{	
  	//返回NULL
    return NULL;
}

IJKFF_Pipeline *ffpipeline_create_from_ffplay(FFPlayer *ffp)
{
    IJKFF_Pipeline *pipeline = ffpipeline_alloc(&g_pipeline_class, sizeof(IJKFF_Pipeline_Opaque));
    if (!pipeline)
        return pipeline;

    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    opaque->ffp                   = ffp;

    pipeline->func_destroy            = func_destroy;
    pipeline->func_open_video_decoder = func_open_video_decoder;
  	//在这里
    pipeline->func_open_audio_output  = func_open_audio_output;

    return pipeline;
}
```

这个平台无关的函数赋值语句赋值的函数返回了NULL。而我再全局搜索一下这个`ffpipeline_create_from_ffplay`，发现没有调用这个的地方。那么这个函数应该只是一个示范函数，让Android和ios平台去各自实现。

那么看下android这边的：

``` c
IJKFF_Pipeline *ffpipeline_create_from_android(FFPlayer *ffp)
{
    ALOGD("ffpipeline_create_from_android()\n");
    IJKFF_Pipeline *pipeline = ffpipeline_alloc(&g_pipeline_class, sizeof(IJKFF_Pipeline_Opaque));
    if (!pipeline)
        return pipeline;

    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    opaque->ffp                   = ffp;
    opaque->surface_mutex         = SDL_CreateMutex();
    opaque->left_volume           = 1.0f;
    opaque->right_volume          = 1.0f;
    if (!opaque->surface_mutex) {
        ALOGE("ffpipeline-android:create SDL_CreateMutex failed\n");
        goto fail;
    }

    pipeline->func_destroy              = func_destroy;
    pipeline->func_open_video_decoder   = func_open_video_decoder;
  	//打开音频输出设备
    pipeline->func_open_audio_output    = func_open_audio_output;
    pipeline->func_init_video_decoder   = func_init_video_decoder;
    pipeline->func_config_video_decoder = func_config_video_decoder;

    return pipeline;
fail:
    ffpipeline_free_p(&pipeline);
    return NULL;
}
```

注意，这个`ffpipeline_create_from_android`方法被调用的地方，是在创建完`ffplayer`播放器后，和`ffplayer->vout`一起创建的，在`3.3.2`中有示例代码。

继续看`func_open_audio_output`函数：

``` c
static SDL_Aout *func_open_audio_output(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    SDL_Aout *aout = NULL;
    if (ffp->opensles) {
      	//如果打开了opensles，则用OpenSLES来创建音频输出设备
        aout = SDL_AoutAndroid_CreateForOpenSLES();
    } else {
      	//否则，使用Android平台的AudioTrack来创建音频输出设备
        aout = SDL_AoutAndroid_CreateForAudioTrack();
    }
    if (aout)
        SDL_AoutSetStereoVolume(aout, pipeline->opaque->left_volume, pipeline->opaque->right_volume);
    return aout;
}
```

那么这个`ffp->opensles`的返回值就很关键了，通过全局搜索，我看到在

`inline static void ffp_reset_internal(*FFPlayer* **ffp*)`函数中有：

``` c
    ffp->opensles                       = 0; // option
```

即opensles是默认关闭的，除非用了option去打开它。

而option的定义位于：`ijkmedia/ijkplayer/ff_ffplay_options.h`。option是如何发挥作用的，后面再分析。



对于ijkplayer是如何利用AudioTrack来播放解码后的pcm音频数据的，这里也暂不分析。

### 3 打开流

```c
    VideoState *is = stream_open(ffp, file_name, NULL);
```

单看这一句，感觉是：根据file_name(url)打开对应的视频流，并返回一个`VideoState`(视频状态)

而这个`VideoState`是保存在`FFPlayer`里面的

``` c
typedef struct FFPlayer {
    const AVClass *av_class;

    /* ffplay context */
    VideoState *is;
  	//...
}
```

而这个`FFPlayer`则是`IikMediaPlayer`中真正的播放器对象。

即一个播放器对应一个`VideoState`对象。

那么先看一下`VideoState`的结构体：

``` c
typedef struct VideoState {
    SDL_Thread *read_tid;//读线程
    SDL_Thread _read_tid;
    AVInputFormat *iformat;//输入格式
    int abort_request;//停止请求
    int force_refresh;//强制刷新
    int paused;//暂停
    int last_paused;
    int queue_attachments_req;
    int seek_req;
    int seek_flags;
    int64_t seek_pos;
    int64_t seek_rel;
#ifdef FFP_MERGE
    int read_pause_return;
#endif
    AVFormatContext *ic;
    int realtime;

    Clock audclk;//音频时钟
    Clock vidclk;//视频时钟
    Clock extclk;//外部时钟

    FrameQueue pictq;//图片帧队列：解码后的视频数据
    FrameQueue subpq;//字幕帧队列：解码后的字幕数据
    FrameQueue sampq;//音频帧队列：解码后的音频数据

    Decoder auddec;//音频解码器
    Decoder viddec;//视频解码器
    Decoder subdec;//字幕解码器

    int audio_stream;//音频流

    int av_sync_type;
    void *handle;
    double audio_clock;
    int audio_clock_serial;
    double audio_diff_cum; /* used for AV difference average computation */
    double audio_diff_avg_coef;
    double audio_diff_threshold;
    int audio_diff_avg_count;
    AVStream *audio_st;
    PacketQueue audioq;//音频包数据：未解码的音频数据，从demuxers输出
    int audio_hw_buf_size;
    uint8_t *audio_buf;
    uint8_t *audio_buf1;
    short *audio_new_buf;  /* for soundtouch buf */
    unsigned int audio_buf_size; /* in bytes */
    unsigned int audio_buf1_size;
    unsigned int audio_new_buf_size;
    int audio_buf_index; /* in bytes */
    int audio_write_buf_size;
    int audio_volume;
    int muted;
    struct AudioParams audio_src;
#if CONFIG_AVFILTER
    struct AudioParams audio_filter_src;
#endif
    struct AudioParams audio_tgt;
    struct SwrContext *swr_ctx;
    int frame_drops_early;
    int frame_drops_late;
    int continuous_frame_drops_early;

    enum ShowMode {
        SHOW_MODE_NONE = -1, SHOW_MODE_VIDEO = 0, SHOW_MODE_WAVES, SHOW_MODE_RDFT, SHOW_MODE_NB
    } show_mode;
    int16_t sample_array[SAMPLE_ARRAY_SIZE];
    int sample_array_index;
    int last_i_start;
#ifdef FFP_MERGE
    RDFTContext *rdft;
    int rdft_bits;
    FFTSample *rdft_data;
    int xpos;
#endif
    double last_vis_time;
#ifdef FFP_MERGE
    SDL_Texture *vis_texture;
    SDL_Texture *sub_texture;
#endif

    int subtitle_stream;
    AVStream *subtitle_st;
    PacketQueue subtitleq;//未解码的字幕数据：从demuxser输出

    double frame_timer;
    double frame_last_returned_time;
    double frame_last_filter_delay;
    int video_stream;
    AVStream *video_st;
    PacketQueue videoq;//未解码的视频数据：从demuxsers输出
    double max_frame_duration;      // maximum duration of a frame - above this, we consider the jump a timestamp discontinuity
    struct SwsContext *img_convert_ctx;
#ifdef FFP_SUB
    struct SwsContext *sub_convert_ctx;
#endif
    int eof;

    char *filename;
    int width, height, xleft, ytop;//视频的：宽、高、左上角x坐标，左上角y坐标。和ffmpeg里面的是对应的。
    int step;

#if CONFIG_AVFILTER
    int vfilter_idx;
    AVFilterContext *in_video_filter;   // the first filter in the video chain
    AVFilterContext *out_video_filter;  // the last filter in the video chain
    AVFilterContext *in_audio_filter;   // the first filter in the audio chain
    AVFilterContext *out_audio_filter;  // the last filter in the audio chain
    AVFilterGraph *agraph;              // audio filter graph
#endif

    int last_video_stream, last_audio_stream, last_subtitle_stream;

    SDL_cond *continue_read_thread;

    /* extra fields */
    SDL_mutex  *play_mutex; // only guard state, do not block any long operation
    SDL_Thread *video_refresh_tid;
    SDL_Thread _video_refresh_tid;

    int buffering_on;
    int pause_req;

    int dropping_frame;
    int is_video_high_fps; // above 30fps
    int is_video_high_res; // above 1080p

    PacketQueue *buffer_indicator_queue;

    volatile int latest_video_seek_load_serial;
    volatile int latest_audio_seek_load_serial;
    volatile int64_t latest_seek_load_start_at;

    int drop_aframe_count;
    int drop_vframe_count;
    int64_t accurate_seek_start_time;
    volatile int64_t accurate_seek_vframe_pts;
    volatile int64_t accurate_seek_aframe_pts;
    int audio_accurate_seek_req;
    int video_accurate_seek_req;
    SDL_mutex *accurate_seek_mutex;
    SDL_cond  *video_accurate_seek_cond;
    SDL_cond  *audio_accurate_seek_cond;
    volatile int initialized_decoder;
    int seek_buffering;
} VideoState;
```

我针对我理解了的字段做了一些注释。



那么现在看到返回`VideoState`结构体的方法`openstream`

``` c
static VideoState *stream_open(FFPlayer *ffp, const char *filename, AVInputFormat *iformat)
{
    assert(!ffp->is);
    VideoState *is;
    //创建VideoState结构体
    is = av_mallocz(sizeof(VideoState));
    if (!is)
        return NULL;
    //给VideoState结构体中的属性赋值
    is->filename = av_strdup(filename);
    if (!is->filename)
        goto fail;
    is->iformat = iformat;
    is->ytop    = 0;
    is->xleft   = 0;
#if defined(__ANDROID__)
    //android平台下的soundtouch，不太清楚是做什么的
    if (ffp->soundtouch_enable) {
        is->handle = ijk_soundtouch_create();
    }
#endif

    /* start video display */
    //初始化3个帧队列（解码后帧的队列）
    if (frame_queue_init(&is->pictq, &is->videoq, ffp->pictq_size, 1) < 0)
        goto fail;
    if (frame_queue_init(&is->subpq, &is->subtitleq, SUBPICTURE_QUEUE_SIZE, 0) < 0)
        goto fail;
    if (frame_queue_init(&is->sampq, &is->audioq, SAMPLE_QUEUE_SIZE, 1) < 0)
        goto fail;
    //初始化3个包队列（解码前的帧队列,不过是demuxer输出的数据了）
    if (packet_queue_init(&is->videoq) < 0 ||
        packet_queue_init(&is->audioq) < 0 ||
        packet_queue_init(&is->subtitleq) < 0)
        goto fail;

    //以下3个创建SDL_cond的函数，不太清楚他们的作用是什么，暂不分析
    if (!(is->continue_read_thread = SDL_CreateCond())) {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        goto fail;
    }

    if (!(is->video_accurate_seek_cond = SDL_CreateCond())) {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        ffp->enable_accurate_seek = 0;
    }

    if (!(is->audio_accurate_seek_cond = SDL_CreateCond())) {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        ffp->enable_accurate_seek = 0;
    }
    //初始化音频时钟，视频时钟，外部时钟
    init_clock(&is->vidclk, &is->videoq.serial);
    init_clock(&is->audclk, &is->audioq.serial);
    init_clock(&is->extclk, &is->extclk.serial);
    is->audio_clock_serial = -1;
    //初始化播放器的初始音量
    if (ffp->startup_volume < 0)
        av_log(NULL, AV_LOG_WARNING, "-volume=%d < 0, setting to 0\n", ffp->startup_volume);
    if (ffp->startup_volume > 100)
        av_log(NULL, AV_LOG_WARNING, "-volume=%d > 100, setting to 100\n", ffp->startup_volume);
    ffp->startup_volume = av_clip(ffp->startup_volume, 0, 100);
    ffp->startup_volume = av_clip(SDL_MIX_MAXVOLUME * ffp->startup_volume / 100, 0, SDL_MIX_MAXVOLUME);
    is->audio_volume = ffp->startup_volume;
    is->muted = 0;
    is->av_sync_type = ffp->av_sync_type;
    //初始化播放器互斥锁
    is->play_mutex = SDL_CreateMutex();
    is->accurate_seek_mutex = SDL_CreateMutex();
    ffp->is = is;
    //如果start_on_prepared=false，那么当prepare完之后要暂停，不能直接播放。
    is->pause_req = !ffp->start_on_prepared;
    //创建视频渲染线程
    is->video_refresh_tid = SDL_CreateThreadEx(&is->_video_refresh_tid, video_refresh_thread, ffp, "ff_vout");
    if (!is->video_refresh_tid) {
        av_freep(&ffp->is);
        return NULL;
    }
    //********开始初始化解码器
    is->initialized_decoder = 0;
    //创建读取线程
    is->read_tid = SDL_CreateThreadEx(&is->_read_tid, read_thread, ffp, "ff_read");
    if (!is->read_tid) {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateThread(): %s\n", SDL_GetError());
        goto fail;
    }

    if (ffp->async_init_decoder && !ffp->video_disable && ffp->video_mime_type && strlen(ffp->video_mime_type) > 0
                    && ffp->mediacodec_default_name && strlen(ffp->mediacodec_default_name) > 0) {
        if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2) {
            decoder_init(&is->viddec, NULL, &is->videoq, is->continue_read_thread);
            ffp->node_vdec = ffpipeline_init_video_decoder(ffp->pipeline, ffp);
        }
    }
    //********初始化解码器完成
    is->initialized_decoder = 1;

    return is;
fail:
    is->initialized_decoder = 1;
    is->abort_request = true;
    if (is->video_refresh_tid)
        SDL_WaitThread(is->video_refresh_tid, NULL);
    stream_close(ffp);
    return NULL;
}
```

他的逻辑大致为：

1. 创建`VideoState`对象，并初始化他的一些默认属性。
2. 初始化视频、音频、字幕的解码后的帧队列。
3. 初始化视频、音频、字幕的解码前的包队列。
4. 初始化播放器音量。
5. 创建视频渲染线程。
6. 创建视频数据读取线程（从网络读取或者从文件读取，io操作）。
7. 初始化解码器。（ffmpeg应该会在内部创建解码线程）。

因此，在`openstream()`方法中完成了最主要的3个线程的创建。



### 4 视频读取线程

在`stream_open`这个打开流的函数中，在开启了视频渲染线程后，接着就开启了视频读取线程。

``` c
static VideoState *stream_open(FFPlayer *ffp, const char *filename, AVInputFormat *iformat){
  	//...
		is->read_tid = SDL_CreateThreadEx(&is->_read_tid, read_thread, ffp, "ff_read");
  	//...
}
```

通过创建单独的线程，专门用于读取Packet，在`read_thread()`函数中。而这个函数非常长，做了很多事情，先放出一个浓缩版的：

#### 简略版代码：

``` c
static int read_thread(void *arg)
{
   	//Open an input stream and read the header. The codecs are not opened.
    //The stream must be closed with avformat_close_input().
    //打开输入流，并读取文件头部，解码器还未打开。主要作用是探测流的协议，如http还是rtmp等。
    err = avformat_open_input(&ic, is->filename, is->iformat, &ffp->format_opts);
  
    // Read packets of a media file to get stream information. This
    // is useful for file formats with no headers such as MPEG. This
    // function also computes the real framerate in case of MPEG-2 repeat
    // frame mode.
    // The logical file position is not changed by this function;
    // examined packets may buffered for later processing. 
    //探测文件封装格式，音视频编码参数等信息。
    err = avformat_find_stream_info(ic, opts);
  
    // Find the "best" stream in the file.
    // The best stream is determined according to various heuristics as the most
    // likely to be what the user expects.
    // If the decoder parameter is non-NULL, av_find_best_stream will find the
    // default decoder for the stream's codec; streams for which no decoder can
    // be found are ignored.
    //根据 AVFormatContext，找到最佳的流。
    av_find_best_stream(ic, AVMEDIA_TYPE_VIDEO,
                        st_index[AVMEDIA_TYPE_VIDEO], -1, NULL, 0);
  
  	//内部分别开启audio,video,subtitle的解码器的线程，开始各自的解码的工作。在稍后的3.6.5解码线程中分析这里的内容
    stream_component_open(ffp, st_index[AVMEDIA_TYPE_AUDIO]);
    stream_component_open(ffp, st_index[AVMEDIA_TYPE_VIDEO]);
    stream_component_open(ffp, st_index[AVMEDIA_TYPE_SUBTITLE]);

  	//开始无限循环,调用ffmpeg的av_read_frame()读取AVPacket，并入队。
	  for (;;) {
      //AVPacket pkt;
      
      ret = av_read_frame(ic, pkt);
      
      //把网络读取到并解封装到的pkt包入队列。（稍后在解码线程会拿到这些pkt包去解码。）
      
			//如果是音频流的包
      packet_queue_put(&is->audioq, pkt);
      //如果是视频流的包
      packet_queue_put(&is->videoq, pkt);
      //如果是字幕流的包
      packet_queue_put(&is->subtitleq, pkt);
    }  
}
```



#### 完整版代码：

那么详细的全部的源码（600行）如下，做了部分注释。

``` c
/* this thread gets the stream from the disk or the network */
static int read_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFormatContext *ic = NULL;
    int err, i, ret __unused;
    int st_index[AVMEDIA_TYPE_NB];
    AVPacket pkt1, *pkt = &pkt1;
    int64_t stream_start_time;
    int completed = 0;
    int pkt_in_play_range = 0;
    AVDictionaryEntry *t;
    SDL_mutex *wait_mutex = SDL_CreateMutex();
    int scan_all_pmts_set = 0;
    int64_t pkt_ts;
    int last_error = 0;
    int64_t prev_io_tick_counter = 0;
    int64_t io_tick_counter = 0;
    int init_ijkmeta = 0;

    if (!wait_mutex) {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateMutex(): %s\n", SDL_GetError());
        ret = AVERROR(ENOMEM);
        goto fail;
    }

    memset(st_index, -1, sizeof(st_index));
    is->last_video_stream = is->video_stream = -1;
    is->last_audio_stream = is->audio_stream = -1;
    is->last_subtitle_stream = is->subtitle_stream = -1;
    is->eof = 0;
    //初始化AVFormatContext
    ic = avformat_alloc_context();
    if (!ic) {
        av_log(NULL, AV_LOG_FATAL, "Could not allocate context.\n");
        ret = AVERROR(ENOMEM);
        goto fail;
    }
    //为AVFormatContext设置中断回调
    ic->interrupt_callback.callback = decode_interrupt_cb;
    ic->interrupt_callback.opaque = is;
    if (!av_dict_get(ffp->format_opts, "scan_all_pmts", NULL, AV_DICT_MATCH_CASE)) {
        av_dict_set(&ffp->format_opts, "scan_all_pmts", "1", AV_DICT_DONT_OVERWRITE);
        scan_all_pmts_set = 1;
    }
    if (av_stristart(is->filename, "rtmp", NULL) ||
        av_stristart(is->filename, "rtsp", NULL)) {
        // There is total different meaning for 'timeout' option in rtmp
        av_log(ffp, AV_LOG_WARNING, "remove 'timeout' option for rtmp.\n");
        av_dict_set(&ffp->format_opts, "timeout", NULL, 0);
    }

    if (ffp->skip_calc_frame_rate) {
        av_dict_set_int(&ic->metadata, "skip-calc-frame-rate", ffp->skip_calc_frame_rate, 0);
        av_dict_set_int(&ffp->format_opts, "skip-calc-frame-rate", ffp->skip_calc_frame_rate, 0);
    }

    if (ffp->iformat_name)
        //找到视频格式:AVInputFormat
        is->iformat = av_find_input_format(ffp->iformat_name);
    //Open an input stream and read the header. The codecs are not opened.
    //The stream must be closed with avformat_close_input().
    //打开输入流，并读取文件头部，解码器还未打开。主要作用是探测流的协议，如http还是rtmp等。
    err = avformat_open_input(&ic, is->filename, is->iformat, &ffp->format_opts);
    if (err < 0) {
        print_error(is->filename, err);
        ret = -1;
        goto fail;
    }
    ffp_notify_msg1(ffp, FFP_MSG_OPEN_INPUT);

    if (scan_all_pmts_set)
        av_dict_set(&ffp->format_opts, "scan_all_pmts", NULL, AV_DICT_MATCH_CASE);

    if ((t = av_dict_get(ffp->format_opts, "", NULL, AV_DICT_IGNORE_SUFFIX))) {
        av_log(NULL, AV_LOG_ERROR, "Option %s not found.\n", t->key);
#ifdef FFP_MERGE
        ret = AVERROR_OPTION_NOT_FOUND;
        goto fail;
#endif
    }
    is->ic = ic;

    if (ffp->genpts)
        ic->flags |= AVFMT_FLAG_GENPTS;

    av_format_inject_global_side_data(ic);
    //
    //AVDictionary **opts;
    //int orig_nb_streams;
    //opts = setup_find_stream_info_opts(ic, ffp->codec_opts);
    //orig_nb_streams = ic->nb_streams;


    if (ffp->find_stream_info) {
        AVDictionary **opts = setup_find_stream_info_opts(ic, ffp->codec_opts);
        int orig_nb_streams = ic->nb_streams;

        do {
            if (av_stristart(is->filename, "data:", NULL) && orig_nb_streams > 0) {
                for (i = 0; i < orig_nb_streams; i++) {
                    if (!ic->streams[i] || !ic->streams[i]->codecpar || ic->streams[i]->codecpar->profile == FF_PROFILE_UNKNOWN) {
                        break;
                    }
                }

                if (i == orig_nb_streams) {
                    break;
                }
            }
            // Read packets of a media file to get stream information. This
            // is useful for file formats with no headers such as MPEG. This
            // function also computes the real framerate in case of MPEG-2 repeat
            // frame mode.
            // The logical file position is not changed by this function;
            // examined packets may buffered for later processing. 
            //探测文件封装格式，音视频编码参数等信息。
            err = avformat_find_stream_info(ic, opts);
        } while(0);
        ffp_notify_msg1(ffp, FFP_MSG_FIND_STREAM_INFO);

        for (i = 0; i < orig_nb_streams; i++)
            av_dict_free(&opts[i]);
        av_freep(&opts);

        if (err < 0) {
            av_log(NULL, AV_LOG_WARNING,
                   "%s: could not find codec parameters\n", is->filename);
            ret = -1;
            goto fail;
        }
    }
    if (ic->pb)
        ic->pb->eof_reached = 0; // FIXME hack, ffplay maybe should not use avio_feof() to test for the end

    if (ffp->seek_by_bytes < 0)
        ffp->seek_by_bytes = !!(ic->iformat->flags & AVFMT_TS_DISCONT) && strcmp("ogg", ic->iformat->name);

    is->max_frame_duration = (ic->iformat->flags & AVFMT_TS_DISCONT) ? 10.0 : 3600.0;
    is->max_frame_duration = 10.0;
    av_log(ffp, AV_LOG_INFO, "max_frame_duration: %.3f\n", is->max_frame_duration);

#ifdef FFP_MERGE
    if (!window_title && (t = av_dict_get(ic->metadata, "title", NULL, 0)))
        window_title = av_asprintf("%s - %s", t->value, input_filename);

#endif
    //处理seek
    /* if seeking requested, we execute it */
    if (ffp->start_time != AV_NOPTS_VALUE) {
        int64_t timestamp;

        timestamp = ffp->start_time;
        /* add the stream start time */
        if (ic->start_time != AV_NOPTS_VALUE)
            timestamp += ic->start_time;
        ret = avformat_seek_file(ic, -1, INT64_MIN, timestamp, INT64_MAX, 0);
        if (ret < 0) {
            av_log(NULL, AV_LOG_WARNING, "%s: could not seek to position %0.3f\n",
                    is->filename, (double)timestamp / AV_TIME_BASE);
        }
    }

    is->realtime = is_realtime(ic);
    //打印详细的格式信息
    av_dump_format(ic, 0, is->filename, 0);

    int video_stream_count = 0;
    int h264_stream_count = 0;
    int first_h264_stream = -1;
    for (i = 0; i < ic->nb_streams; i++) {
        AVStream *st = ic->streams[i];
        enum AVMediaType type = st->codecpar->codec_type;
        st->discard = AVDISCARD_ALL;
        if (type >= 0 && ffp->wanted_stream_spec[type] && st_index[type] == -1)
            if (avformat_match_stream_specifier(ic, st, ffp->wanted_stream_spec[type]) > 0)
                st_index[type] = i;

        // choose first h264

        if (type == AVMEDIA_TYPE_VIDEO) {
            enum AVCodecID codec_id = st->codecpar->codec_id;
            video_stream_count++;
            if (codec_id == AV_CODEC_ID_H264) {
                h264_stream_count++;
                if (first_h264_stream < 0)
                    first_h264_stream = i;
            }
        }
    }
    if (video_stream_count > 1 && st_index[AVMEDIA_TYPE_VIDEO] < 0) {
        st_index[AVMEDIA_TYPE_VIDEO] = first_h264_stream;
        av_log(NULL, AV_LOG_WARNING, "multiple video stream found, prefer first h264 stream: %d\n", first_h264_stream);
    }

    //*****对视频，音频，和字幕，调用av_find_best_stream，找到对应的stream的下标，并填充在st_index数组中
    if (!ffp->video_disable)
        st_index[AVMEDIA_TYPE_VIDEO] =
            // Find the "best" stream in the file.
            // The best stream is determined according to various heuristics as the most
            // likely to be what the user expects.
            // If the decoder parameter is non-NULL, av_find_best_stream will find the
            // default decoder for the stream's codec; streams for which no decoder can
            // be found are ignored.
            //根据AVFormatContext，找到最佳的流。
            av_find_best_stream(ic, AVMEDIA_TYPE_VIDEO,
                                st_index[AVMEDIA_TYPE_VIDEO], -1, NULL, 0);
    if (!ffp->audio_disable)
        st_index[AVMEDIA_TYPE_AUDIO] =
            av_find_best_stream(ic, AVMEDIA_TYPE_AUDIO,
                                st_index[AVMEDIA_TYPE_AUDIO],
                                st_index[AVMEDIA_TYPE_VIDEO],
                                NULL, 0);
    if (!ffp->video_disable && !ffp->subtitle_disable)
        st_index[AVMEDIA_TYPE_SUBTITLE] =
            av_find_best_stream(ic, AVMEDIA_TYPE_SUBTITLE,
                                st_index[AVMEDIA_TYPE_SUBTITLE],
                                (st_index[AVMEDIA_TYPE_AUDIO] >= 0 ?
                                 st_index[AVMEDIA_TYPE_AUDIO] :
                                 st_index[AVMEDIA_TYPE_VIDEO]),
                                NULL, 0);

    is->show_mode = ffp->show_mode;
#ifdef FFP_MERGE // bbc: dunno if we need this
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
        AVStream *st = ic->streams[st_index[AVMEDIA_TYPE_VIDEO]];
        AVCodecParameters *codecpar = st->codecpar;
        AVRational sar = av_guess_sample_aspect_ratio(ic, st, NULL);
        if (codecpar->width)
            set_default_window_size(codecpar->width, codecpar->height, sar);
    }
#endif

    //******打开3个流 start
    /* open the streams */
    if (st_index[AVMEDIA_TYPE_AUDIO] >= 0) {
        stream_component_open(ffp, st_index[AVMEDIA_TYPE_AUDIO]);
    } else {
        ffp->av_sync_type = AV_SYNC_VIDEO_MASTER;
        is->av_sync_type  = ffp->av_sync_type;
    }

    ret = -1;
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
        ret = stream_component_open(ffp, st_index[AVMEDIA_TYPE_VIDEO]);
    }
    if (is->show_mode == SHOW_MODE_NONE)
        is->show_mode = ret >= 0 ? SHOW_MODE_VIDEO : SHOW_MODE_RDFT;

    if (st_index[AVMEDIA_TYPE_SUBTITLE] >= 0) {
        stream_component_open(ffp, st_index[AVMEDIA_TYPE_SUBTITLE]);
    }
    //******打开3个流 end

    ffp_notify_msg1(ffp, FFP_MSG_COMPONENT_OPEN);

    if (!ffp->ijkmeta_delay_init) {
        ijkmeta_set_avformat_context_l(ffp->meta, ic);
    }

    ffp->stat.bit_rate = ic->bit_rate;
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0)
        ijkmeta_set_int64_l(ffp->meta, IJKM_KEY_VIDEO_STREAM, st_index[AVMEDIA_TYPE_VIDEO]);
    if (st_index[AVMEDIA_TYPE_AUDIO] >= 0)
        ijkmeta_set_int64_l(ffp->meta, IJKM_KEY_AUDIO_STREAM, st_index[AVMEDIA_TYPE_AUDIO]);
    if (st_index[AVMEDIA_TYPE_SUBTITLE] >= 0)
        ijkmeta_set_int64_l(ffp->meta, IJKM_KEY_TIMEDTEXT_STREAM, st_index[AVMEDIA_TYPE_SUBTITLE]);

    if (is->video_stream < 0 && is->audio_stream < 0) {
        av_log(NULL, AV_LOG_FATAL, "Failed to open file '%s' or configure filtergraph\n",
               is->filename);
        ret = -1;
        goto fail;
    }

    //初始化缓冲指示器队列，优先使用音频packetQueue，找不到就使用视频packetQueue。
    if (is->audio_stream >= 0) {
        is->audioq.is_buffer_indicator = 1;
        is->buffer_indicator_queue = &is->audioq;
    } else if (is->video_stream >= 0) {
        is->videoq.is_buffer_indicator = 1;
        is->buffer_indicator_queue = &is->videoq;
    } else {
        assert("invalid streams");
    }

    //如果是直播流，则播放器使用无限缓存
    if (ffp->infinite_buffer < 0 && is->realtime)
        ffp->infinite_buffer = 1;

    if (!ffp->render_wait_start && !ffp->start_on_prepared)
        toggle_pause(ffp, 1);
    if (is->video_st && is->video_st->codecpar) {
        AVCodecParameters *codecpar = is->video_st->codecpar;
        //发送VIDEO_SIZE_CHANGED回调和SAR_CHANGED回调
        ffp_notify_msg3(ffp, FFP_MSG_VIDEO_SIZE_CHANGED, codecpar->width, codecpar->height);
        ffp_notify_msg3(ffp, FFP_MSG_SAR_CHANGED, codecpar->sample_aspect_ratio.num, codecpar->sample_aspect_ratio.den);
    }
    ffp->prepared = true;
    //发送PREPARED回调
    ffp_notify_msg1(ffp, FFP_MSG_PREPARED);
    if (!ffp->render_wait_start && !ffp->start_on_prepared) {
        while (is->pause_req && !is->abort_request) {
            SDL_Delay(20);
        }
    }
    if (ffp->auto_resume) {
        ffp_notify_msg1(ffp, FFP_REQ_START);
        ffp->auto_resume = 0;
    }
    /* offset should be seeked*/
    if (ffp->seek_at_start > 0) {
        ffp_seek_to_l(ffp, (long)(ffp->seek_at_start));
    }

    //开始无限循环,调用ffmpeg的av_read_frame()读取AVPacket，并入队。
    for (;;) {
        //如果中断请求，则跳出循环
        if (is->abort_request)
            break;
#ifdef FFP_MERGE
        if (is->paused != is->last_paused) {
            is->last_paused = is->paused;
            if (is->paused)
                is->read_pause_return = av_read_pause(ic);
            else
                av_read_play(ic);
        }
#endif
#if CONFIG_RTSP_DEMUXER || CONFIG_MMSH_PROTOCOL
        if (is->paused &&
                (!strcmp(ic->iformat->name, "rtsp") ||
                 (ic->pb && !strncmp(ffp->input_filename, "mmsh:", 5)))) {
            /* wait 10 ms to avoid trying to get another packet */
            /* XXX: horrible */
            SDL_Delay(10);
            continue;
        }
#endif
        //如果是seek 请求
        if (is->seek_req) {
            int64_t seek_target = is->seek_pos;
            int64_t seek_min    = is->seek_rel > 0 ? seek_target - is->seek_rel + 2: INT64_MIN;
            int64_t seek_max    = is->seek_rel < 0 ? seek_target - is->seek_rel - 2: INT64_MAX;
// FIXME the +-2 is due to rounding being not done in the correct direction in generation
//      of the seek_pos/seek_rel variables

            ffp_toggle_buffering(ffp, 1);
            ffp_notify_msg3(ffp, FFP_MSG_BUFFERING_UPDATE, 0, 0);
            //ffmepg 中处理seek
            ret = avformat_seek_file(is->ic, -1, seek_min, seek_target, seek_max, is->seek_flags);
            if (ret < 0) {
                av_log(NULL, AV_LOG_ERROR,
                       "%s: error while seeking\n", is->ic->filename);
            } else {
                if (is->audio_stream >= 0) {
                    packet_queue_flush(&is->audioq);
                    packet_queue_put(&is->audioq, &flush_pkt);
                    // TODO: clear invaild audio data
                    // SDL_AoutFlushAudio(ffp->aout);
                }
                if (is->subtitle_stream >= 0) {
                    packet_queue_flush(&is->subtitleq);
                    packet_queue_put(&is->subtitleq, &flush_pkt);
                }
                if (is->video_stream >= 0) {
                    if (ffp->node_vdec) {
                        ffpipenode_flush(ffp->node_vdec);
                    }
                    packet_queue_flush(&is->videoq);
                    packet_queue_put(&is->videoq, &flush_pkt);
                }
                if (is->seek_flags & AVSEEK_FLAG_BYTE) {
                   set_clock(&is->extclk, NAN, 0);
                } else {
                   set_clock(&is->extclk, seek_target / (double)AV_TIME_BASE, 0);
                }

                is->latest_video_seek_load_serial = is->videoq.serial;
                is->latest_audio_seek_load_serial = is->audioq.serial;
                is->latest_seek_load_start_at = av_gettime();
            }
            ffp->dcc.current_high_water_mark_in_ms = ffp->dcc.first_high_water_mark_in_ms;
            is->seek_req = 0;
            is->queue_attachments_req = 1;
            is->eof = 0;
#ifdef FFP_MERGE
            if (is->paused)
                step_to_next_frame(is);
#endif
            completed = 0;
            SDL_LockMutex(ffp->is->play_mutex);
            if (ffp->auto_resume) {
                is->pause_req = 0;
                if (ffp->packet_buffering)
                    is->buffering_on = 1;
                ffp->auto_resume = 0;
                stream_update_pause_l(ffp);
            }
            if (is->pause_req)
                step_to_next_frame_l(ffp);
            SDL_UnlockMutex(ffp->is->play_mutex);

            if (ffp->enable_accurate_seek) {
                is->drop_aframe_count = 0;
                is->drop_vframe_count = 0;
                SDL_LockMutex(is->accurate_seek_mutex);
                if (is->video_stream >= 0) {
                    is->video_accurate_seek_req = 1;
                }
                if (is->audio_stream >= 0) {
                    is->audio_accurate_seek_req = 1;
                }
                SDL_CondSignal(is->audio_accurate_seek_cond);
                SDL_CondSignal(is->video_accurate_seek_cond);
                SDL_UnlockMutex(is->accurate_seek_mutex);
            }

            ffp_notify_msg3(ffp, FFP_MSG_SEEK_COMPLETE, (int)fftime_to_milliseconds(seek_target), ret);
            ffp_toggle_buffering(ffp, 1);
        }
        if (is->queue_attachments_req) {
            if (is->video_st && (is->video_st->disposition & AV_DISPOSITION_ATTACHED_PIC)) {
                AVPacket copy = { 0 };
                if ((ret = av_packet_ref(&copy, &is->video_st->attached_pic)) < 0)
                    goto fail;
                packet_queue_put(&is->videoq, &copy);
                packet_queue_put_nullpacket(&is->videoq, is->video_stream);
            }
            is->queue_attachments_req = 0;
        }

        /* if the queue are full, no need to read more */
        if (ffp->infinite_buffer<1 && !is->seek_req &&
#ifdef FFP_MERGE
              (is->audioq.size + is->videoq.size + is->subtitleq.size > MAX_QUEUE_SIZE
#else
              (is->audioq.size + is->videoq.size + is->subtitleq.size > ffp->dcc.max_buffer_size
#endif
                    //内部逻辑为queue->nb_packets > min_frames
            || (   stream_has_enough_packets(is->audio_st, is->audio_stream, &is->audioq, MIN_FRAMES)
                && stream_has_enough_packets(is->video_st, is->video_stream, &is->videoq, MIN_FRAMES)
                && stream_has_enough_packets(is->subtitle_st, is->subtitle_stream, &is->subtitleq, MIN_FRAMES)))) {
            if (!is->eof) {
                ffp_toggle_buffering(ffp, 0);
            }
            /* wait 10 ms */
            SDL_LockMutex(wait_mutex);
            SDL_CondWaitTimeout(is->continue_read_thread, wait_mutex, 10);
            SDL_UnlockMutex(wait_mutex);
            //进入到下一次循环
            continue;
        }
        //处理播放结束
        if ((!is->paused || completed) &&
            (!is->audio_st || (is->auddec.finished == is->audioq.serial && frame_queue_nb_remaining(&is->sampq) == 0)) &&
            (!is->video_st || (is->viddec.finished == is->videoq.serial && frame_queue_nb_remaining(&is->pictq) == 0))) {
            if (ffp->loop != 1 && (!ffp->loop || --ffp->loop)) {
                stream_seek(is, ffp->start_time != AV_NOPTS_VALUE ? ffp->start_time : 0, 0, 0);
            } else if (ffp->autoexit) {
                ret = AVERROR_EOF;
                goto fail;
            } else {
                ffp_statistic_l(ffp);
                if (completed) {
                    av_log(ffp, AV_LOG_INFO, "ffp_toggle_buffering: eof\n");
                    SDL_LockMutex(wait_mutex);
                    // infinite wait may block shutdown
                    while(!is->abort_request && !is->seek_req)
                        SDL_CondWaitTimeout(is->continue_read_thread, wait_mutex, 100);
                    SDL_UnlockMutex(wait_mutex);
                    if (!is->abort_request)
                        continue;
                } else {
                    completed = 1;
                    ffp->auto_resume = 0;

                    // TODO: 0 it's a bit early to notify complete here
                    ffp_toggle_buffering(ffp, 0);
                    toggle_pause(ffp, 1);
                    if (ffp->error) {
                        av_log(ffp, AV_LOG_INFO, "ffp_toggle_buffering: error: %d\n", ffp->error);
                        ffp_notify_msg1(ffp, FFP_MSG_ERROR);
                    } else {
                        av_log(ffp, AV_LOG_INFO, "ffp_toggle_buffering: completed: OK\n");
                        ffp_notify_msg1(ffp, FFP_MSG_COMPLETED);
                    }
                }
            }
        }
        pkt->flags = 0;
        //读帧，读到这个pkt包里面？
        //0 if OK, < 0 on error or end of file
        ret = av_read_frame(ic, pkt);
        if (ret < 0) {
            int pb_eof = 0;
            int pb_error = 0;
            //EOF表示：end of file
            if ((ret == AVERROR_EOF || avio_feof(ic->pb)) && !is->eof) {
                ffp_check_buffering_l(ffp);
                pb_eof = 1;
                // check error later
            }
            if (ic->pb && ic->pb->error) {
                pb_eof = 1;
                pb_error = ic->pb->error;
            }
            if (ret == AVERROR_EXIT) {
                pb_eof = 1;
                pb_error = AVERROR_EXIT;
            }

            if (pb_eof) {
                if (is->video_stream >= 0)
                    packet_queue_put_nullpacket(&is->videoq, is->video_stream);
                if (is->audio_stream >= 0)
                    packet_queue_put_nullpacket(&is->audioq, is->audio_stream);
                if (is->subtitle_stream >= 0)
                    packet_queue_put_nullpacket(&is->subtitleq, is->subtitle_stream);
                is->eof = 1;
            }
            if (pb_error) {
                if (is->video_stream >= 0)
                    packet_queue_put_nullpacket(&is->videoq, is->video_stream);
                if (is->audio_stream >= 0)
                    packet_queue_put_nullpacket(&is->audioq, is->audio_stream);
                if (is->subtitle_stream >= 0)
                    packet_queue_put_nullpacket(&is->subtitleq, is->subtitle_stream);
                is->eof = 1;
                ffp->error = pb_error;
                av_log(ffp, AV_LOG_ERROR, "av_read_frame error: %s\n", ffp_get_error_string(ffp->error));
                // break;
            } else {
                ffp->error = 0;
            }
            if (is->eof) {
                ffp_toggle_buffering(ffp, 0);
                SDL_Delay(100);
            }
            SDL_LockMutex(wait_mutex);
            SDL_CondWaitTimeout(is->continue_read_thread, wait_mutex, 10);
            SDL_UnlockMutex(wait_mutex);
            ffp_statistic_l(ffp);
            continue;
        } else {
            is->eof = 0;
        }

        //flush_pkt是用来做什么的？
        if (pkt->flags & AV_PKT_FLAG_DISCONTINUITY) {
            if (is->audio_stream >= 0) {
                packet_queue_put(&is->audioq, &flush_pkt);
            }
            if (is->subtitle_stream >= 0) {
                packet_queue_put(&is->subtitleq, &flush_pkt);
            }
            if (is->video_stream >= 0) {
                packet_queue_put(&is->videoq, &flush_pkt);
            }
        }

        /* check if packet is in play range specified by user, then queue, otherwise discard */
        stream_start_time = ic->streams[pkt->stream_index]->start_time;
        pkt_ts = pkt->pts == AV_NOPTS_VALUE ? pkt->dts : pkt->pts;
        pkt_in_play_range = ffp->duration == AV_NOPTS_VALUE ||
                (pkt_ts - (stream_start_time != AV_NOPTS_VALUE ? stream_start_time : 0)) *
                av_q2d(ic->streams[pkt->stream_index]->time_base) -
                (double)(ffp->start_time != AV_NOPTS_VALUE ? ffp->start_time : 0) / 1000000
                <= ((double)ffp->duration / 1000000);
        if (pkt->stream_index == is->audio_stream && pkt_in_play_range) {
            packet_queue_put(&is->audioq, pkt);
        } else if (pkt->stream_index == is->video_stream && pkt_in_play_range
                   && !(is->video_st && (is->video_st->disposition & AV_DISPOSITION_ATTACHED_PIC))) {
            packet_queue_put(&is->videoq, pkt);
        } else if (pkt->stream_index == is->subtitle_stream && pkt_in_play_range) {
            packet_queue_put(&is->subtitleq, pkt);
        } else {
            av_packet_unref(pkt);
        }

        ffp_statistic_l(ffp);

        if (ffp->ijkmeta_delay_init && !init_ijkmeta &&
                (ffp->first_video_frame_rendered || !is->video_st) && (ffp->first_audio_frame_rendered || !is->audio_st)) {
            ijkmeta_set_avformat_context_l(ffp->meta, ic);
            init_ijkmeta = 1;
        }

        if (ffp->packet_buffering) {
            io_tick_counter = SDL_GetTickHR();
            if ((!ffp->first_video_frame_rendered && is->video_st) || (!ffp->first_audio_frame_rendered && is->audio_st)) {
                if (abs((int)(io_tick_counter - prev_io_tick_counter)) > FAST_BUFFERING_CHECK_PER_MILLISECONDS) {
                    prev_io_tick_counter = io_tick_counter;
                    ffp->dcc.current_high_water_mark_in_ms = ffp->dcc.first_high_water_mark_in_ms;
                    ffp_check_buffering_l(ffp);
                }
            } else {
                if (abs((int)(io_tick_counter - prev_io_tick_counter)) > BUFFERING_CHECK_PER_MILLISECONDS) {
                    prev_io_tick_counter = io_tick_counter;
                    ffp_check_buffering_l(ffp);
                }
            }
        }
    }

    ret = 0;
 fail:
    if (ic && !is->ic)
        avformat_close_input(&ic);

    if (!ffp->prepared || !is->abort_request) {
        ffp->last_error = last_error;
        ffp_notify_msg2(ffp, FFP_MSG_ERROR, last_error);
    }
    SDL_DestroyMutex(wait_mutex);
    return 0;
}
```



#### 小结：

总结一下：视频读取线程大致做的事情就是：

1. ffmpeg进行协议探测，封装格式探测等网络请求，为创建解码器做准备。
2. 创建video，audio，subtitle解码器并开启相应的解码线程。
3. for循环不断地调用`av_read_frame()`去从ffmpeg内部维护的网络包缓存去取出下载好的AVPacket，并放入相应的队列中，供稍后解码线程取出解码。



