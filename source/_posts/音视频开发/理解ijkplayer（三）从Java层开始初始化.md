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



## 1. 应用层使用

``` kotlin
//实例化
val videoView:IjkVideoView = IjkVideoView(this)
//添加到布局
fl_video_container.addView(
    videoView,
    ViewGroup.LayoutParams.MATCH_PARENT,
    ViewGroup.LayoutParams.MATCH_PARENT
)
//设置uri
videoView.setVideoURI(Uri.parse(url))
//开始播放
videoView.start()
```

短短4行代码就集成了一个播放器，实在太简单了。

现在开始分析。



## 2. 初始化

``` kotlin
val videoView:IjkVideoView = IjkVideoView(this)
```

构造函数中调用了-->

``` java
private void initVideoView(Context context) {
    mAppContext = context.getApplicationContext();
    mSettings = new Settings(mAppContext);
    //是否开启后台播放，如果开启了后台播放，就启动一个Service来做后台播放。
    initBackground();
    //初始化渲染器，创建SurfaceView或TextureView，并addView()。（IjkVideoView是FrameLayout）
    initRenders();
    //初始化播放器宽高
    mVideoWidth = 0;
    mVideoHeight = 0;
    // REMOVED: getHolder().addCallback(mSHCallback);
    // REMOVED: getHolder().setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
    
    //焦点
    setFocusable(true);
    setFocusableInTouchMode(true);
    requestFocus();
    // REMOVED: mPendingSubtitleTracks = new Vector<Pair<InputStream, MediaFormat>>();
    
    //状态
    mCurrentState = STATE_IDLE;
    mTargetState = STATE_IDLE;
    //字幕View
    subtitleDisplay = new TextView(context);
    subtitleDisplay.setTextSize(24);
    subtitleDisplay.setGravity(Gravity.CENTER);
    FrameLayout.LayoutParams layoutParams_txt = new FrameLayout.LayoutParams(
            FrameLayout.LayoutParams.MATCH_PARENT,
            FrameLayout.LayoutParams.WRAP_CONTENT,
            Gravity.BOTTOM);
    addView(subtitleDisplay, layoutParams_txt);
}
```



注意：

`initRenders()`创建的渲染器View，要在`IjkMediaPlayer`对象创建后才设置给播放器对象，现在只是创建，并添加到layout。



## 3. setVideoURI()

``` java
private void setVideoURI(Uri uri, Map<String, String> headers) {
    mUri = uri;
    mHeaders = headers;
    mSeekWhenPrepared = 0;
  	//打开视频
    openVideo();
    requestLayout();
    invalidate();
}
```



``` java
@TargetApi(Build.VERSION_CODES.M)
private void openVideo() {
    if (mUri == null || mSurfaceHolder == null) {
        // not ready for playback just yet, will try again later
        return;
    }
    // we shouldn't clear the target state, because somebody might have
    // called start() previously
    release(false);
  
  	//获取音频焦点
    AudioManager am = (AudioManager) mAppContext.getSystemService(Context.AUDIO_SERVICE);
    am.requestAudioFocus(null, AudioManager.STREAM_MUSIC, AudioManager.AUDIOFOCUS_GAIN);
    try {
      	//创建IjkMediaPlayer对象，他是真正的播放器对象。
        mMediaPlayer = createPlayer(mSettings.getPlayer());
        // TODO: create SubtitleController in MediaPlayer, but we need
        // a context for the subtitle renderers
        final Context context = getContext();
        // REMOVED: SubtitleController
        // REMOVED: mAudioSession
      	
      	//设置IjkMediaPlayer对象从c层发出的各种播放器事件的回调，并通过对应的mXXXListener变量转发到上层
        mMediaPlayer.setOnPreparedListener(mPreparedListener);
        mMediaPlayer.setOnVideoSizeChangedListener(mSizeChangedListener);
        mMediaPlayer.setOnCompletionListener(mCompletionListener);
        mMediaPlayer.setOnErrorListener(mErrorListener);
        mMediaPlayer.setOnInfoListener(mInfoListener);
        mMediaPlayer.setOnBufferingUpdateListener(mBufferingUpdateListener);
        mMediaPlayer.setOnSeekCompleteListener(mSeekCompleteListener);
        mMediaPlayer.setOnTimedTextListener(mOnTimedTextListener);
        mCurrentBufferPercentage = 0;
        String scheme = mUri.getScheme();
      
      	//调用setDataSouce()设置数据源。本地url或者网络url
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M &&
                mSettings.getUsingMediaDataSource() &&
                (TextUtils.isEmpty(scheme) || scheme.equalsIgnoreCase("file"))) {
            IMediaDataSource dataSource = new FileMediaDataSource(new File(mUri.toString()));
            mMediaPlayer.setDataSource(dataSource);
        }  else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            mMediaPlayer.setDataSource(mAppContext, mUri, mHeaders);
        } else {
            mMediaPlayer.setDataSource(mUri.toString());
        }
      	//将渲染器（SurfaceView或TextureView）传递给IjkMediaPlayer对象，即设置渲染器。
        bindSurfaceHolder(mMediaPlayer, mSurfaceHolder);
      	//设置音频流类型（原生MediaPlayer有逻辑，而IjkMediaPlayer是空实现）
        mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
      	//setKeepScreenOn不息屏
        mMediaPlayer.setScreenOnWhilePlaying(true);
      	//记录preapare前的时间（后面onPrepared回调里面会根据这个事件算出parepare花了多长时间）
        mPrepareStartTime = System.currentTimeMillis();
      	//异步prepare
        mMediaPlayer.prepareAsync();
        if (mHudViewHolder != null)
            mHudViewHolder.setMediaPlayer(mMediaPlayer);
        // REMOVED: mPendingSubtitleTracks
        // we don't set the target state here either, but preserve the
        // target state that was there before.
        mCurrentState = STATE_PREPARING;
        attachMediaController();
    } catch (IOException ex) {
        Log.w(TAG, "Unable to open content: " + mUri, ex);
        mCurrentState = STATE_ERROR;
        mTargetState = STATE_ERROR;
        mErrorListener.onError(mMediaPlayer, MediaPlayer.MEDIA_ERROR_UNKNOWN, 0);
    } catch (IllegalArgumentException ex) {
        Log.w(TAG, "Unable to open content: " + mUri, ex);
        mCurrentState = STATE_ERROR;
        mTargetState = STATE_ERROR;
        mErrorListener.onError(mMediaPlayer, MediaPlayer.MEDIA_ERROR_UNKNOWN, 0);
    } finally {
        // REMOVED: mPendingSubtitleTracks.clear();
    }
}
```

这里创建播放器的入口：

``` java
//创建IjkMediaPlayer对象，他是真正的播放器对象。
mMediaPlayer = createPlayer(mSettings.getPlayer());

//-->ijkMediaPlayer = new IjkMediaPlayer();
```

``` java
//IjkMediaPlayer.java
public IjkMediaPlayer() {
    this(sLocalLibLoader);
}
public IjkMediaPlayer(IjkLibLoader libLoader) {
    initPlayer(libLoader);
}
private void initPlayer(IjkLibLoader libLoader) {
  	//装载so库
    loadLibrariesOnce(libLoader);
  	//第一次调用c层函数，native_init()，初始化c层播放器
    initNativeOnce();
    Looper looper;
    if ((looper = Looper.myLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else if ((looper = Looper.getMainLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else {
        mEventHandler = null;
    }
    /*
     * Native setup requires a weak reference to our object. It's easier to
     * create it here than in C++.
     */
  	//调用native_setup()
    native_setup(new WeakReference<IjkMediaPlayer>(this));
}

public static void loadLibrariesOnce(IjkLibLoader libLoader) {
    synchronized (IjkMediaPlayer.class) {
        if (!mIsLibLoaded) {
            if (libLoader == null)
                libLoader = sLocalLibLoader;
            libLoader.loadLibrary("ijkffmpeg");
            libLoader.loadLibrary("ijksdl");
            libLoader.loadLibrary("ijkplayer");
            mIsLibLoaded = true;
        }
    }
}

private static void initNativeOnce() {
    synchronized (IjkMediaPlayer.class) {
        if (!mIsNativeInitialized) {
            native_init();
            mIsNativeInitialized = true;
        }
    }
}
```



那么，在`setVideoURI()`方法中，总共调用了jni的方法为：

| java方法                                         | jni方法                   |
| ------------------------------------------------ | ------------------------- |
| initNativeOnce                                   | native_init()             |
| initPlayer()中                                   | native_setup              |
| setDataSource()                                  | _setDataSource()          |
| setDisplay(SurfaceHolder) /  setSurface(Surface) | _setVideoSurface(Surface) |
| prepareAsync()                                   | _prepareAsync()           |



接下来逐个分析各个c层的方法做了什么。

在开始分析c层代码之前，要知道一下ijkplayer的jni方法是动态注册的。

动态注册Jni方法详见[官方文档](https://developer.android.google.cn/training/articles/perf-jni)

即借助`JNI_OnLoad()`方法去动态注册。

### 3.1 JNI_Onload()

``` c
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
    JNIEnv* env = NULL;

    g_jvm = vm;
    if ((*vm)->GetEnv(vm, (void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        return -1;
    }
    assert(env != NULL);

    pthread_mutex_init(&g_clazz.mutex, NULL );

    // FindClass returns LocalReference
    IJK_FIND_JAVA_CLASS(env, g_clazz.clazz, JNI_CLASS_IJKPLAYER);
    //标准的饿RegisterNatives方法，g_methods方法返回要注册的jni方法数组
    (*env)->RegisterNatives(env, g_clazz.clazz, g_methods, NELEM(g_methods) );

    //播放器全局初始化，注册ffmpeg的解码器，解封装器，加载外部库如openssl等
    ijkmp_global_init();
    ijkmp_global_set_inject_callback(inject_callback);

    FFmpegApi_global_init(env);

    return JNI_VERSION_1_4;
}
```



#### 3.1.1 注册jni方法

在`g_methods()`方法中，定义了所有的从java-->c的jni方法。

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



#### 3.1.2 播放器全局初始化

后面调用

``` c
//播放器全局初始化，注册ffmpeg的解码器，解封装器，加载外部库如openssl等
ijkmp_global_init();
```

调用到`ijkmedia/ijkplayer/ff_ffplay.c`

``` c
void ffp_global_init()
{
    if (g_ffmpeg_global_inited)
        return;

    ALOGD("ijkmediaplayer version : %s", ijkmp_version());
    /* register all codecs, demux and protocols */
    //ffmpeg方法，注册所有的编码器和解码器
    avcodec_register_all();
#if CONFIG_AVDEVICE
    //ffmpeg方法，注册所有设备
    avdevice_register_all();
#endif
#if CONFIG_AVFILTER
    //注册所有过滤器（注册所有过滤器）
    avfilter_register_all();
#endif
    //ffmpeg方法，注册所有封装器和解封装器
    av_register_all();

    //ijkmedia/ijkplayer/ijkavformat/allformats.c注册ijkplayer额外支持的协议，例如ijkio，async，ijklongurl
    ijkav_register_all();

    //初始化openssl库
    avformat_network_init();

    av_lockmgr_register(lockmgr);
    av_log_set_callback(ffp_log_callback_brief);

    av_init_packet(&flush_pkt);
    flush_pkt.data = (uint8_t *)&flush_pkt;

    g_ffmpeg_global_inited = true;
}
```

到这里，ijkplayer借助`JNI_OnLoad（）`方法进行了

1. jni方法的注册
2. ffmpeg编解码器的注册，ffmpeg封装解封装器的注册，其他外部库如openssl的加载等。



### 3.2 native_init()

根据`g_methods()方法`中的jni方法映射，找到对应的方法，所有直接映射的方法都依然在`ijkmedia/ijkplayer/android/ijkplayer_jni.c`中定义（后续所有的Jni方法直接给出对应的c层映射方法）：

``` c
static void
IjkMediaPlayer_native_init(JNIEnv *env)
{
    MPTRACE("%s\n", __func__);
}
```

这个`MPTRACE()`方法会去调用`ALOG()`方法，而后者调用`(void)printf(__VA_ARGS__)`，也就是打印。

我猜这个方法应该是作者预留给开发者的初始化方法？



### 3.3 native_setup()

``` c
static void
IjkMediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    MPTRACE("%s\n", __func__);
    //创建IjkMediaPlayer，并传入message_loop()方法作为参数
    IjkMediaPlayer *mp = ijkmp_android_create(message_loop);
    JNI_CHECK_GOTO(mp, env, "java/lang/OutOfMemoryError", "mpjni: native_setup: ijkmp_create() failed", LABEL_RETURN);

    jni_set_media_player(env, thiz, mp);
    ijkmp_set_weak_thiz(mp, (*env)->NewGlobalRef(env, weak_this));
    ijkmp_set_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_set_ijkio_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_android_set_mediacodec_select_callback(mp, mediacodec_select_callback, ijkmp_get_weak_thiz(mp));

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```





#### 3.3.1 创建类似looper的消息机制

在上面的代码中，`message_loop()`方法创建了类似android的looper的消息机制，如果不熟悉的话，要复习一下Android的Looper、MessageQueue、Handler那一套了。

``` c
static int message_loop(void *arg)
{
    MPTRACE("%s\n", __func__);

    JNIEnv *env = NULL;
    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s: SetupThreadEnv failed\n", __func__);
        return -1;
    }

    IjkMediaPlayer *mp = (IjkMediaPlayer*) arg;
    JNI_CHECK_GOTO(mp, env, NULL, "mpjni: native_message_loop: null mp", LABEL_RETURN);
    //开启类似Android的looper的消息机制。
    message_loop_n(env, mp);

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);

    MPTRACE("message_loop exit");
    return 0;
}
```



``` c
//message_loop_n函数中取消息，并用post_event调用jni方法把事件发送到java层
static void message_loop_n(JNIEnv *env, IjkMediaPlayer *mp)
{
    jobject weak_thiz = (jobject) ijkmp_get_weak_thiz(mp);
    JNI_CHECK_GOTO(weak_thiz, env, NULL, "mpjni: message_loop_n: null weak_thiz", LABEL_RETURN);

    while (1) {
        AVMessage msg;
        //取消息队列的消息，如果没有消息就阻塞，直到有消息被发到消息队列。
        int retval = ijkmp_get_msg(mp, &msg, 1);
        if (retval < 0)
            break;

        // block-get should never return 0
        assert(retval > 0);

        switch (msg.what) {
        case FFP_MSG_FLUSH:
            MPTRACE("FFP_MSG_FLUSH:\n");
            //调用post_event，把事件发送到java层。
            post_event(env, weak_thiz, MEDIA_NOP, 0, 0);
            break;
        case FFP_MSG_ERROR:
            MPTRACE("FFP_MSG_ERROR: %d\n", msg.arg1);
            post_event(env, weak_thiz, MEDIA_ERROR, MEDIA_ERROR_IJK_PLAYER, msg.arg1);
            break;
        case FFP_MSG_PREPARED:
            MPTRACE("FFP_MSG_PREPARED:\n");
            post_event(env, weak_thiz, MEDIA_PREPARED, 0, 0);
            break;
        case FFP_MSG_COMPLETED:
            MPTRACE("FFP_MSG_COMPLETED:\n");
            post_event(env, weak_thiz, MEDIA_PLAYBACK_COMPLETE, 0, 0);
            break;
        //...其他省略
        msg_free_res(&msg);
    }

LABEL_RETURN:
    ;
}
```



``` c
//获取消息，并在这个函数中针对每一个取出来的消息做c层的处理，例如ijkplayer的当前播放器状态的赋值。
int ijkmp_get_msg(IjkMediaPlayer *mp, AVMessage *msg, int block)
{
    assert(mp);
    while (1) {
        int continue_wait_next_msg = 0;
        //取消息，如果没有消息则阻塞。
        int retval = msg_queue_get(&mp->ffplayer->msg_queue, msg, block);
        if (retval <= 0)
            return retval;

        switch (msg->what) {
        case FFP_MSG_PREPARED:
            MPTRACE("ijkmp_get_msg: FFP_MSG_PREPARED\n");
            pthread_mutex_lock(&mp->mutex);
            if (mp->mp_state == MP_STATE_ASYNC_PREPARING) {
                ijkmp_change_state_l(mp, MP_STATE_PREPARED);
            } else {
                // FIXME: 1: onError() ?
                av_log(mp->ffplayer, AV_LOG_DEBUG, "FFP_MSG_PREPARED: expecting mp_state==MP_STATE_ASYNC_PREPARING\n");
            }
            if (!mp->ffplayer->start_on_prepared) {
                ijkmp_change_state_l(mp, MP_STATE_PAUSED);
            }
            pthread_mutex_unlock(&mp->mutex);
            break;

        case FFP_MSG_COMPLETED:
            MPTRACE("ijkmp_get_msg: FFP_MSG_COMPLETED\n");

            pthread_mutex_lock(&mp->mutex);
            mp->restart = 1;
            mp->restart_from_beginning = 1;
            ijkmp_change_state_l(mp, MP_STATE_COMPLETED);
            pthread_mutex_unlock(&mp->mutex);
            break;
        //...省略
        if (continue_wait_next_msg) {
            msg_free_res(msg);
            continue;
        }

        return retval;
    }

    return -1;
}

```



``` c
/* return < 0 if aborted, 0 if no msg and > 0 if msg.  */
inline static int msg_queue_get(MessageQueue *q, AVMessage *msg, int block)
{
    AVMessage *msg1;
    int ret;

    SDL_LockMutex(q->mutex);

    for (;;) {
        if (q->abort_request) {
            ret = -1;
            break;
        }

        msg1 = q->first_msg;
        if (msg1) {
            q->first_msg = msg1->next;
            if (!q->first_msg)
                q->last_msg = NULL;
            q->nb_messages--;
            *msg = *msg1;
            msg1->obj = NULL;
#ifdef FFP_MERGE
            av_free(msg1);
#else
            msg1->next = q->recycle_msg;
            q->recycle_msg = msg1;
#endif
            ret = 1;
            break;
        } else if (!block) {
            ret = 0;
            break;
        } else {
            //如果消息队列为空，则阻塞当前线程并等待被唤醒。
            SDL_CondWait(q->cond, q->mutex);
        }
    }
    SDL_UnlockMutex(q->mutex);
    return ret;
}
```



而`post_event（）`方法会将事件从c层抛到java层的：

``` java
//tv.danmaku.ijk.media.player.IjkMediaPlayer

/*
 * Called from native code when an interesting event happens. This method
 * just uses the EventHandler system to post the event back to the main app
 * thread. We use a weak reference to the original IjkMediaPlayer object so
 * that the native code is safe from the object disappearing from underneath
 * it. (This is the cookie passed to native_setup().)
 */
@CalledByNative
private static void postEventFromNative(Object weakThiz, int what,
        int arg1, int arg2, Object obj) {
    if (weakThiz == null)
        return;
    @SuppressWarnings("rawtypes")
    IjkMediaPlayer mp = (IjkMediaPlayer) ((WeakReference) weakThiz).get();
    if (mp == null) {
        return;
    }
    if (what == MEDIA_INFO && arg1 == MEDIA_INFO_STARTED_AS_NEXT) {
        // this acquires the wakelock if needed, and sets the client side
        // state
        mp.start();
    }
    if (mp.mEventHandler != null) {
        Message m = mp.mEventHandler.obtainMessage(what, arg1, arg2, obj);
        mp.mEventHandler.sendMessage(m);
    }
}

private static class EventHandler extends Handler {
        private final WeakReference<IjkMediaPlayer> mWeakPlayer;

        public EventHandler(IjkMediaPlayer mp, Looper looper) {
            super(looper);
            mWeakPlayer = new WeakReference<IjkMediaPlayer>(mp);
        }

        @Override
        public void handleMessage(Message msg) {
            IjkMediaPlayer player = mWeakPlayer.get();
            if (player == null || player.mNativeMediaPlayer == 0) {
                DebugLog.w(TAG,
                        "IjkMediaPlayer went away with unhandled events");
                return;
            }
						//根据事件类型，回调对应的OnXXX方法，把播放器状态回调给使用IjkPlayer的java开发者
            switch (msg.what) {
            case MEDIA_PREPARED:
                player.notifyOnPrepared();
                return;

            case MEDIA_PLAYBACK_COMPLETE:
                player.stayAwake(false);
                player.notifyOnCompletion();
                return;

            case MEDIA_BUFFERING_UPDATE:
                long bufferPosition = msg.arg1;
                if (bufferPosition < 0) {
                    bufferPosition = 0;
                }

                long percent = 0;
                long duration = player.getDuration();
                if (duration > 0) {
                    percent = bufferPosition * 100 / duration;
                }
                if (percent >= 100) {
                    percent = 100;
                }

                // DebugLog.efmt(TAG, "Buffer (%d%%) %d/%d",  percent, bufferPosition, duration);
                player.notifyOnBufferingUpdate((int)percent);
                return;

            case MEDIA_SEEK_COMPLETE:
                player.notifyOnSeekComplete();
                return;

            case MEDIA_SET_VIDEO_SIZE:
                player.mVideoWidth = msg.arg1;
                player.mVideoHeight = msg.arg2;
                player.notifyOnVideoSizeChanged(player.mVideoWidth, player.mVideoHeight,
                        player.mVideoSarNum, player.mVideoSarDen);
                return;

            case MEDIA_ERROR:
                DebugLog.e(TAG, "Error (" + msg.arg1 + "," + msg.arg2 + ")");
                if (!player.notifyOnError(msg.arg1, msg.arg2)) {
                    player.notifyOnCompletion();
                }
                player.stayAwake(false);
                return;

            case MEDIA_INFO:
                switch (msg.arg1) {
                    case MEDIA_INFO_VIDEO_RENDERING_START:
                        DebugLog.i(TAG, "Info: MEDIA_INFO_VIDEO_RENDERING_START\n");
                        break;
                }
                player.notifyOnInfo(msg.arg1, msg.arg2);
                // No real default action so far.
                return;
            case MEDIA_TIMED_TEXT:
                if (msg.obj == null) {
                    player.notifyOnTimedText(null);
                } else {
                    IjkTimedText text = new IjkTimedText(new Rect(0, 0, 1, 1), (String)msg.obj);
                    player.notifyOnTimedText(text);
                }
                return;
            case MEDIA_NOP: // interface test message - ignore
                break;

            case MEDIA_SET_VIDEO_SAR:
                player.mVideoSarNum = msg.arg1;
                player.mVideoSarDen = msg.arg2;
                player.notifyOnVideoSizeChanged(player.mVideoWidth, player.mVideoHeight,
                        player.mVideoSarNum, player.mVideoSarDen);
                break;

            default:
                DebugLog.e(TAG, "Unknown message type " + msg.what);
            }
        }
    }
```

到这里，从java到c和从c到java的事件通信和传送大概就是这些了，然而种类面涉及到一些细节例如：线程转换等，并没有去探究，



注意,`message_loop()`方法是创建一个消息循环机制，那么回到`message_loop()`函数被使用的地方：

``` c
static void
IjkMediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    MPTRACE("%s\n", __func__);
    //创建IjkMediaPlayer，并传入message_loop()方法作为参数
    IjkMediaPlayer *mp = ijkmp_android_create(message_loop);
    JNI_CHECK_GOTO(mp, env, "java/lang/OutOfMemoryError", "mpjni: native_setup: ijkmp_create() failed", LABEL_RETURN);

    jni_set_media_player(env, thiz, mp);
    ijkmp_set_weak_thiz(mp, (*env)->NewGlobalRef(env, weak_this));
    ijkmp_set_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_set_ijkio_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_android_set_mediacodec_select_callback(mp, mediacodec_select_callback, ijkmp_get_weak_thiz(mp));

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```

`message_loop`这个函数作为参数被传递，在这里依然还没有调用，即没有开启循环读队列。那么猜想`message_loop`函数一定是被传递下去，在某个地方被调用了，并且需要在一个独立的线程中调用，就像模拟Android的`HandlerThread`类做的那样，一个独立的线程中开启了循环。



#### 3.3.2 创建IjkMediaPlayer

依然看到上面的代码块，通过`ijkmp_android_create(message_loop)`方法创建了`IjkMediaPlayer`播放器结构体，在看具体创建代码之前，我们先看一下结构体：

``` c
//	ijkmedia/ijkplayer/ijkplayer_internal.h
struct IjkMediaPlayer {
    volatile int ref_count;
  	//互斥锁，用于保证线程安全
    pthread_mutex_t mutex;
  	//ffplayer，位于ijkmedia/ijkplayer/ff_ffplay_def.h，他会直接调用ffmpeg的方法了
    FFPlayer *ffplayer;
		
  	//一个函数指针，指向的是谁？指向的其实就是刚才创建的message_loop，即消息循环函数
    int (*msg_loop)(void*);
  	//消息机制线程
    SDL_Thread *msg_thread;
    SDL_Thread _msg_thread;
		//播放器状态，例如prepared,resumed,error,completed等
    int mp_state;
  	//字符串，就是一个播放url
    char *data_source;
  	//弱引用
    void *weak_thiz;

    int restart;
    int restart_from_beginning;
    int seek_req;
    long seek_msec;
};
```

结构体的说明如注释所示。

那么现在看创建播放器的函数：

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

那么看到创建`IjkMediaPlayer`的函数

``` c
//	ijkmedia/ijkplayer/ff_ffplay.c
IjkMediaPlayer *ijkmp_create(int (*msg_loop)(void*))
{
    //分配内存
    IjkMediaPlayer *mp = (IjkMediaPlayer *) mallocz(sizeof(IjkMediaPlayer));
    if (!mp)
        goto fail;
    //创建IjkMediaPlayer内部的FFPlayer
    mp->ffplayer = ffp_create();
    if (!mp->ffplayer)
        goto fail;
    //注意：将msg_loop函数赋值给IjkMediaPlayer的函数引用，在创建的时候赋值，在另一处被调用。
    //在哪里被调用呢？在prepareAsync()里面，后面分析prepare方法的时候就会再见到消息循环函数了。
    mp->msg_loop = msg_loop;

    ijkmp_inc_ref(mp);
    pthread_mutex_init(&mp->mutex, NULL);

    return mp;

    fail:
    ijkmp_destroy_p(&mp);
    return NULL;
}
```

那么再看到`ffp_create()`方法中创建`FFPlayer`的逻辑

``` c
FFPlayer *ffp_create()
{
    av_log(NULL, AV_LOG_INFO, "av_version_info: %s\n", av_version_info());
    av_log(NULL, AV_LOG_INFO, "ijk_version_info: %s\n", ijk_version_info());

    FFPlayer* ffp = (FFPlayer*) av_mallocz(sizeof(FFPlayer));
    if (!ffp)
        return NULL;
    //创建消息循环的消息队列MessageQueue，这个MessageQueue就是后面在message_loop函数中调用的那个
    msg_queue_init(&ffp->msg_queue);
    ffp->af_mutex = SDL_CreateMutex();
    ffp->vf_mutex = SDL_CreateMutex();
    //重置ffplayer属性
    ffp_reset_internal(ffp);
    //赋值AVClass，是一个常量
    ffp->av_class = &ffp_context_class;
    //创建IjkMediaMeta
    ffp->meta = ijkmeta_create();

    av_opt_set_defaults(ffp);

    return ffp;
}
```

播放器的创建到这里就结束了，但是还没有分析

`ffplay->vout`视频设备

`ffplay->pipeline `管道

这两个东西的创建和作用（因为我自己也还不知道这是干嘛用的，回头再补吧）



### 3.4 _setDataSource()

``` c
//	ijkmedia/ijkplayer/ijkplayer.c
static int ijkmp_set_data_source_l(IjkMediaPlayer *mp, const char *url)
{
    assert(mp);
    assert(url);

    // MPST_RET_IF_EQ(mp->mp_state, MP_STATE_IDLE);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_INITIALIZED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_ASYNC_PREPARING);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_PREPARED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_STARTED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_PAUSED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_COMPLETED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_STOPPED);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_ERROR);
    MPST_RET_IF_EQ(mp->mp_state, MP_STATE_END);

    freep((void**)&mp->data_source);
    //设置Url
    mp->data_source = strdup(url);
    if (!mp->data_source)
        return EIJK_OUT_OF_MEMORY;
    //改变播放器状态为MP_STATE_INITIALIZED
    ijkmp_change_state_l(mp, MP_STATE_INITIALIZED);
    return 0;
}
```

 看一下改变播放器状态的函数：

``` c
//	ijkmedia/ijkplayer/ijkplayer.c
void ijkmp_change_state_l(IjkMediaPlayer *mp, int new_state)
{
    mp->mp_state = new_state;
  	//出现了notify字样，一般这个notify字样的方法，都是表示将某个事件通知外部的监听器
    ffp_notify_msg1(mp->ffplayer, FFP_MSG_PLAYBACK_STATE_CHANGED);
}
```

``` c
//	ijkmedia/ijkplayer/ff_ffmsg_queue.h
inline static void msg_queue_put_simple3(MessageQueue *q, int what, int arg1, int arg2)
{
    AVMessage msg;
    //初始化一条空的msg
    msg_init_msg(&msg);
    msg.what = what;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    //消息入队，这个队列是ffplay->msg_queue。
    msg_queue_put(q, &msg);
}
```

``` c
//	ijkmedia/ijkplayer/ff_ffmsg_queue.h
inline static int msg_queue_put(MessageQueue *q, AVMessage *msg)
{
    int ret;
    //线程安全保护
    SDL_LockMutex(q->mutex);
    ret = msg_queue_put_private(q, msg);
    SDL_UnlockMutex(q->mutex);

    return ret;
}
```

``` c
inline static int msg_queue_put_private(MessageQueue *q, AVMessage *msg)
{
    AVMessage *msg1;

    if (q->abort_request)
        return -1;

#ifdef FFP_MERGE
    msg1 = av_malloc(sizeof(AVMessage));
#else
    msg1 = q->recycle_msg;
    if (msg1) {
        q->recycle_msg = msg1->next;
        q->recycle_count++;
    } else {
        q->alloc_count++;
        msg1 = av_malloc(sizeof(AVMessage));
    }
#ifdef FFP_SHOW_MSG_RECYCLE
    int total_count = q->recycle_count + q->alloc_count;
    if (!(total_count % 10)) {
        av_log(NULL, AV_LOG_DEBUG, "msg-recycle \t%d + \t%d = \t%d\n", q->recycle_count, q->alloc_count, total_count);
    }
#endif
#endif
    if (!msg1)
        return -1;

    *msg1 = *msg;
    msg1->next = NULL;

    if (!q->last_msg)
        q->first_msg = msg1;
    else
        q->last_msg->next = msg1;
    q->last_msg = msg1;
    q->nb_messages++;
    //唤醒正在等待消息线程msg_thread
    SDL_CondSignal(q->cond);
    return 0;
}
```

notify函数的一系列过程就是类似Android中的Handler生成一个Message对象，然后入队MessageQueue，然后MessageQueue唤醒阻塞的Looper线程。

而这里发出去的消息是`FFP_MSG_PLAYBACK_STATE_CHANGED`

全局搜索这个关键字，可以看到，在`ijkplayer_jni.c`的 `static void message_loop_n(JNIEnv **env*, IjkMediaPlayer **mp*)`函数中：

``` c
 case FFP_MSG_PLAYBACK_STATE_CHANGED:
            break;
```

对该事件没有做任何的处理。



所以，`_setDataSource()`方法给播放器设置了url，然后也没有做什么额外的工作了，设置进去的url，应该会在后面某个时候（在prepare_async）用于请求网络。



### 3.5 _setVideoSurface()



``` c
static void
IjkMediaPlayer_setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface)
{
    MPTRACE("%s\n", __func__);
    IjkMediaPlayer *mp = jni_get_media_player(env, thiz);
    JNI_CHECK_GOTO(mp, env, NULL, "mpjni: setVideoSurface: null mp", LABEL_RETURN);

    ijkmp_android_set_surface(env, mp, jsurface);

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
    return;
}
```

``` c
void ijkmp_android_set_surface_l(JNIEnv *env, IjkMediaPlayer *mp, jobject android_surface)
{
    if (!mp || !mp->ffplayer || !mp->ffplayer->vout)
        return;
    //将Surface与ffplayer->vout关联
    SDL_VoutAndroid_SetAndroidSurface(env, mp->ffplayer->vout, android_surface);
    //将Surface与ffplayer->pipeline关联
    ffpipeline_set_surface(env, mp->ffplayer->pipeline, android_surface);
}
```

对于ffplayer->vout和ffplayer->pipeline的理解还不够，因此无法继续深入地分析了。

不过可以肯定的是，这是个函数是用于将c层的视频渲染器和java层设置进来的Surface（来自SurfaceView或者TextureView）。

