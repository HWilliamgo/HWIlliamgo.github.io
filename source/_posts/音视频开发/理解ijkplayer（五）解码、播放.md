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



## 1 解码线程

### 简略版代码：

解码线程位于：`strem_component_open()`中，简略版如下：

``` c
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
	  AVCodecContext *avctx;//解码器上下文
		AVCodec *codec = NULL;//解码器
    //找到解码器
    codec = avcodec_find_decoder(avctx->codec_id);
  
  	switch (avctx->codec_type) {
    case AVMEDIA_TYPE_AUDIO:
        ret = audio_open(ffp, channel_layout, nb_channels, sample_rate, &is->audio_tgt);
       	//decoder初始化
        decoder_init(&is->auddec, avctx, &is->audioq, is->continue_read_thread);
				//decoder启动，启动audio_thread线程
        if ((ret = decoder_start(&is->auddec, audio_thread, ffp, "ff_audio_dec")) < 0)
            goto out;
        break;
    case AVMEDIA_TYPE_VIDEO:
        //decoder初始化
        decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
        ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
        if (!ffp->node_vdec)
          goto fail;
        //解码器开始
        if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
          goto out;
        break;
      case AVMEDIA_TYPE_SUBTITLE:
        //decoder初始化
        decoder_init(&is->subdec, avctx, &is->subtitleq, is->continue_read_thread);
        //解码器开始
        if ((ret = decoder_start(&is->subdec, subtitle_thread, ffp, "ff_subtitle_dec")) < 0)
            goto out;
        break;
}
```



### 完整版代码：

``` c
/* open a given stream. Return 0 if OK */
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
    VideoState *is = ffp->is;
    AVFormatContext *ic = is->ic;
    AVCodecContext *avctx;//解码器上下文
    AVCodec *codec = NULL;//解码器
    const char *forced_codec_name = NULL;
    AVDictionary *opts = NULL;
    AVDictionaryEntry *t = NULL;
    int sample_rate, nb_channels;
    int64_t channel_layout;
    int ret = 0;
    int stream_lowres = ffp->lowres;

    if (stream_index < 0 || stream_index >= ic->nb_streams)
        return -1;
    avctx = avcodec_alloc_context3(NULL);
    if (!avctx)
        return AVERROR(ENOMEM);
    //将AVCodecParameters中的变量赋值给AVCodecContext
    ret = avcodec_parameters_to_context(avctx, ic->streams[stream_index]->codecpar);
    if (ret < 0)
        goto fail;
    av_codec_set_pkt_timebase(avctx, ic->streams[stream_index]->time_base);
    //找到解码器
    codec = avcodec_find_decoder(avctx->codec_id);

    switch (avctx->codec_type) {
        case AVMEDIA_TYPE_AUDIO   : is->last_audio_stream    = stream_index; forced_codec_name = ffp->audio_codec_name; break;
        case AVMEDIA_TYPE_SUBTITLE: is->last_subtitle_stream = stream_index; forced_codec_name = ffp->subtitle_codec_name; break;
        case AVMEDIA_TYPE_VIDEO   : is->last_video_stream    = stream_index; forced_codec_name = ffp->video_codec_name; break;
        default: break;
    }
    if (forced_codec_name)
        codec = avcodec_find_decoder_by_name(forced_codec_name);
    if (!codec) {
        if (forced_codec_name) av_log(NULL, AV_LOG_WARNING,
                                      "No codec could be found with name '%s'\n", forced_codec_name);
        else                   av_log(NULL, AV_LOG_WARNING,
                                      "No codec could be found with id %d\n", avctx->codec_id);
        ret = AVERROR(EINVAL);
        goto fail;
    }

    avctx->codec_id = codec->id;
    if(stream_lowres > av_codec_get_max_lowres(codec)){
        av_log(avctx, AV_LOG_WARNING, "The maximum value for lowres supported by the decoder is %d\n",
                av_codec_get_max_lowres(codec));
        stream_lowres = av_codec_get_max_lowres(codec);
    }
    av_codec_set_lowres(avctx, stream_lowres);

#if FF_API_EMU_EDGE
    if(stream_lowres) avctx->flags |= CODEC_FLAG_EMU_EDGE;
#endif
    if (ffp->fast)
        avctx->flags2 |= AV_CODEC_FLAG2_FAST;
#if FF_API_EMU_EDGE
    if(codec->capabilities & AV_CODEC_CAP_DR1)
        avctx->flags |= CODEC_FLAG_EMU_EDGE;
#endif

    opts = filter_codec_opts(ffp->codec_opts, avctx->codec_id, ic, ic->streams[stream_index], codec);
    if (!av_dict_get(opts, "threads", NULL, 0))
        av_dict_set(&opts, "threads", "auto", 0);
    if (stream_lowres)
        av_dict_set_int(&opts, "lowres", stream_lowres, 0);
    if (avctx->codec_type == AVMEDIA_TYPE_VIDEO || avctx->codec_type == AVMEDIA_TYPE_AUDIO)
        av_dict_set(&opts, "refcounted_frames", "1", 0);
    if ((ret = avcodec_open2(avctx, codec, &opts)) < 0) {
        goto fail;
    }
    if ((t = av_dict_get(opts, "", NULL, AV_DICT_IGNORE_SUFFIX))) {
        av_log(NULL, AV_LOG_ERROR, "Option %s not found.\n", t->key);
#ifdef FFP_MERGE
        ret =  AVERROR_OPTION_NOT_FOUND;
        goto fail;
#endif
    }

    is->eof = 0;
    ic->streams[stream_index]->discard = AVDISCARD_DEFAULT;
    switch (avctx->codec_type) {
    case AVMEDIA_TYPE_AUDIO:
#if CONFIG_AVFILTER
        {
            AVFilterContext *sink;

            is->audio_filter_src.freq           = avctx->sample_rate;
            is->audio_filter_src.channels       = avctx->channels;
            is->audio_filter_src.channel_layout = get_valid_channel_layout(avctx->channel_layout, avctx->channels);
            is->audio_filter_src.fmt            = avctx->sample_fmt;
            SDL_LockMutex(ffp->af_mutex);
            if ((ret = configure_audio_filters(ffp, ffp->afilters, 0)) < 0) {
                SDL_UnlockMutex(ffp->af_mutex);
                goto fail;
            }
            ffp->af_changed = 0;
            SDL_UnlockMutex(ffp->af_mutex);
            sink = is->out_audio_filter;
            sample_rate    = av_buffersink_get_sample_rate(sink);
            nb_channels    = av_buffersink_get_channels(sink);
            channel_layout = av_buffersink_get_channel_layout(sink);
        }
#else
        sample_rate    = avctx->sample_rate;
        nb_channels    = avctx->channels;
        channel_layout = avctx->channel_layout;
#endif

        /* prepare audio output */
        //audio_open方法是在做什么？
        if ((ret = audio_open(ffp, channel_layout, nb_channels, sample_rate, &is->audio_tgt)) < 0)
            goto fail;
        ffp_set_audio_codec_info(ffp, AVCODEC_MODULE_NAME, avcodec_get_name(avctx->codec_id));
        is->audio_hw_buf_size = ret;
        is->audio_src = is->audio_tgt;
        is->audio_buf_size  = 0;
        is->audio_buf_index = 0;

        /* init averaging filter */
        is->audio_diff_avg_coef  = exp(log(0.01) / AUDIO_DIFF_AVG_NB);
        is->audio_diff_avg_count = 0;
        /* since we do not have a precise anough audio FIFO fullness,
           we correct audio sync only if larger than this threshold */
        is->audio_diff_threshold = 2.0 * is->audio_hw_buf_size / is->audio_tgt.bytes_per_sec;

        is->audio_stream = stream_index;
        is->audio_st = ic->streams[stream_index];
        //decoder初始化
        decoder_init(&is->auddec, avctx, &is->audioq, is->continue_read_thread);
        if ((is->ic->iformat->flags & (AVFMT_NOBINSEARCH | AVFMT_NOGENSEARCH | AVFMT_NO_BYTE_SEEK)) && !is->ic->iformat->read_seek) {
            is->auddec.start_pts = is->audio_st->start_time;
            is->auddec.start_pts_tb = is->audio_st->time_base;
        }
        //decoder启动，启动audio_thread线程
        if ((ret = decoder_start(&is->auddec, audio_thread, ffp, "ff_audio_dec")) < 0)
            goto out;
        SDL_AoutPauseAudio(ffp->aout, 0);
        break;
    case AVMEDIA_TYPE_VIDEO:
        is->video_stream = stream_index;
        is->video_st = ic->streams[stream_index];
        //async_init_decoder是一个option，默认是0
        if (ffp->async_init_decoder) {
            while (!is->initialized_decoder) {
                SDL_Delay(5);
            }
            if (ffp->node_vdec) {
                is->viddec.avctx = avctx;
                ret = ffpipeline_config_video_decoder(ffp->pipeline, ffp);
            }
            if (ret || !ffp->node_vdec) {
                decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
                ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
                if (!ffp->node_vdec)
                    goto fail;
            }
        } else {
            //decoder初始化
            decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
            ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
            if (!ffp->node_vdec)
                goto fail;
        }
        //解码器开始
        if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
            goto out;

        is->queue_attachments_req = 1;

        if (ffp->max_fps >= 0) {
            if(is->video_st->avg_frame_rate.den && is->video_st->avg_frame_rate.num) {
                double fps = av_q2d(is->video_st->avg_frame_rate);
                SDL_ProfilerReset(&is->viddec.decode_profiler, fps + 0.5);
                if (fps > ffp->max_fps && fps < 130.0) {
                    is->is_video_high_fps = 1;
                    av_log(ffp, AV_LOG_WARNING, "fps: %lf (too high)\n", fps);
                } else {
                    av_log(ffp, AV_LOG_WARNING, "fps: %lf (normal)\n", fps);
                }
            }
            if(is->video_st->r_frame_rate.den && is->video_st->r_frame_rate.num) {
                double tbr = av_q2d(is->video_st->r_frame_rate);
                if (tbr > ffp->max_fps && tbr < 130.0) {
                    is->is_video_high_fps = 1;
                    av_log(ffp, AV_LOG_WARNING, "fps: %lf (too high)\n", tbr);
                } else {
                    av_log(ffp, AV_LOG_WARNING, "fps: %lf (normal)\n", tbr);
                }
            }
        }

        if (is->is_video_high_fps) {
            avctx->skip_frame       = FFMAX(avctx->skip_frame, AVDISCARD_NONREF);
            avctx->skip_loop_filter = FFMAX(avctx->skip_loop_filter, AVDISCARD_NONREF);
            avctx->skip_idct        = FFMAX(avctx->skip_loop_filter, AVDISCARD_NONREF);
        }

        break;
    case AVMEDIA_TYPE_SUBTITLE:
        if (!ffp->subtitle) break;

        is->subtitle_stream = stream_index;
        is->subtitle_st = ic->streams[stream_index];

        ffp_set_subtitle_codec_info(ffp, AVCODEC_MODULE_NAME, avcodec_get_name(avctx->codec_id));

        decoder_init(&is->subdec, avctx, &is->subtitleq, is->continue_read_thread);
        if ((ret = decoder_start(&is->subdec, subtitle_thread, ffp, "ff_subtitle_dec")) < 0)
            goto out;
        break;
    default:
        break;
    }
    goto out;

fail:
    avcodec_free_context(&avctx);
out:
    av_dict_free(&opts);

    return ret;
}
```



### 小结：

1. 找到解码器
2. 初始化解码器
3. 分别启动`audio_thread`，`video_thread`和`subtitle_thread`这3条解码线程，内部开始不断解码。



那么以下3节则逐个分析这3条解码线程



## 2 字幕解码线程`subtitle_thread`

由于字幕解码线程最简单，所以先来看看他是如何工作的，对剩下的两个解码线程就更好理解了。

``` c
static int subtitle_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    Frame *sp;
    int got_subtitle;
    double pts;

    for (;;) {
        //阻塞方法，阻塞，直到能取出windex（写下标）下标下的Frame
        if (!(sp = frame_queue_peek_writable(&is->subpq)))
            return 0;
        //解码，填充Frame中的字幕数据
        if ((got_subtitle = decoder_decode_frame(ffp, &is->subdec, NULL, &sp->sub)) < 0)
            break;

        pts = 0;
#ifdef FFP_MERGE
        if (got_subtitle && sp->sub.format == 0) {
#else
        if (got_subtitle) {
#endif
            if (sp->sub.pts != AV_NOPTS_VALUE)
                pts = sp->sub.pts / (double)AV_TIME_BASE;
            sp->pts = pts;
            sp->serial = is->subdec.pkt_serial;
            sp->width = is->subdec.avctx->width;
            sp->height = is->subdec.avctx->height;
            sp->uploaded = 0;

            /* now we can update the picture count */
            //后移字幕FrameQueue的windex
            frame_queue_push(&is->subpq);
#ifdef FFP_MERGE
        } else if (got_subtitle) {
            avsubtitle_free(&sp->sub);
#endif
        }
    }
    return 0;
}
```

解码后的数据`sp`要保存，留着待会渲染，也就是要入队，那么由于`FrameQueue`是数组的特殊性，因此入队的操作不需要新建的frame数据作为参数，只需要确保数组中的write index的数据正确填充，然后将write index后移一个位置，就称为入队成功了：

``` c
static void  frame_queue_push(FrameQueue *f)
{
    //当使用数组作为队列的时候，只需要移动数组中的下标到有效下标，就表示入队了，并不需要外部再传一个参数进来。
    //如果到了尾下标，则windex回到起点。这是用数组作为循环队列的必要操作。
    if (++f->windex == f->max_size)
        f->windex = 0;
    SDL_LockMutex(f->mutex);
    f->size++;
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```



那么接着来看：`decoder_decode_frame（）`

注意：本文基于0.8.0的ijkplayer，这个函数和以前的ijkplayer的解码逻辑和调用的ffmpeg的函数都有些区别。我看到0.8.0的版本的`decoder_decode_frame（）`函数的逻辑是在`0.8.7`的时候修改并上线的。

##### 

### 2.1 `decoder_decode_frame()`，since version 0.8.7

先看本文基于的0.8.0的ijkplayer的函数：

``` c
static int decoder_decode_frame(FFPlayer *ffp, Decoder *d, AVFrame *frame, AVSubtitle *sub) {
    int ret = AVERROR(EAGAIN);

    for (;;) {
        AVPacket pkt;

        if (d->queue->serial == d->pkt_serial) {
            do {
                if (d->queue->abort_request)
                    return -1;

                switch (d->avctx->codec_type) {
                    case AVMEDIA_TYPE_VIDEO:
                        //从解码器中接收frame数据。当返回0表示成功
                        ret = avcodec_receive_frame(d->avctx, frame);
                        if (ret >= 0) {
                            ffp->stat.vdps = SDL_SpeedSamplerAdd(&ffp->vdps_sampler, FFP_SHOW_VDPS_AVCODEC, "vdps[avcodec]");
                            if (ffp->decoder_reorder_pts == -1) {
                                frame->pts = frame->best_effort_timestamp;
                            } else if (!ffp->decoder_reorder_pts) {
                                frame->pts = frame->pkt_dts;
                            }
                        }
                        break;
                    case AVMEDIA_TYPE_AUDIO:
                        //从解码器中接收frame数据。当返回0表示成功
                        ret = avcodec_receive_frame(d->avctx, frame);
                        if (ret >= 0) {
                            AVRational tb = (AVRational){1, frame->sample_rate};
                            if (frame->pts != AV_NOPTS_VALUE)
                                frame->pts = av_rescale_q(frame->pts, av_codec_get_pkt_timebase(d->avctx), tb);
                            else if (d->next_pts != AV_NOPTS_VALUE)
                                frame->pts = av_rescale_q(d->next_pts, d->next_pts_tb, tb);
                            if (frame->pts != AV_NOPTS_VALUE) {
                                d->next_pts = frame->pts + frame->nb_samples;
                                d->next_pts_tb = tb;
                            }
                        }
                        break;
                    default:
                        break;
                }
                if (ret == AVERROR_EOF) {
                    d->finished = d->pkt_serial;
                    avcodec_flush_buffers(d->avctx);
                    return 0;
                }
                //如果返回值>=0，表示avcodec_receive_frame函数解码成功，那么从外部函数decoder_decode_frame返回1。
                //视频，音频，字幕的解码都从这里返回，只要解码成功，都去读取ret然后返回给外面处理。
                if (ret >= 0)
                    return 1;
            } while (ret != AVERROR(EAGAIN));
        }

        do {
            if (d->queue->nb_packets == 0)
                SDL_CondSignal(d->empty_queue_cond);
            if (d->packet_pending) {
                av_packet_move_ref(&pkt, &d->pkt);
                d->packet_pending = 0;
            } else {
                //从packet_queue中取出pkt，当packat_queue由于网络差等原因，没有足够的包可以取出时，则阻塞，直到有包能取出。
                if (packet_queue_get_or_buffering(ffp, d->queue, &pkt, &d->pkt_serial, &d->finished) < 0)
                    return -1;
            }
        } while (d->queue->serial != d->pkt_serial);

        if (pkt.data == flush_pkt.data) {
            avcodec_flush_buffers(d->avctx);
            d->finished = 0;
            d->next_pts = d->start_pts;
            d->next_pts_tb = d->start_pts_tb;
        } else {
            if (d->avctx->codec_type == AVMEDIA_TYPE_SUBTITLE) {
                int got_frame = 0;
                //解码字幕
                ret = avcodec_decode_subtitle2(d->avctx, sub, &got_frame, &pkt);
                if (ret < 0) {
                    ret = AVERROR(EAGAIN);
                } else {
                    if (got_frame && !pkt.data) {
                       d->packet_pending = 1;
                       av_packet_move_ref(&d->pkt, &pkt);
                    }
                    ret = got_frame ? 0 : (pkt.data ? AVERROR(EAGAIN) : AVERROR_EOF);
                }
            } else {
                //往解码器里面发送包数据pkt
                if (avcodec_send_packet(d->avctx, &pkt) == AVERROR(EAGAIN)) {
                    av_log(d->avctx, AV_LOG_ERROR, "Receive_frame and send_packet both returned EAGAIN, which is an API violation.\n");
                    d->packet_pending = 1;
                    av_packet_move_ref(&d->pkt, &pkt);
                }
            }
            av_packet_unref(&pkt);
        }
    }
}
```



### 2.2 `decoder_decode_frame()`，before version 0.8.7



``` c
static int decoder_decode_frame(FFPlayer *ffp, Decoder *d, AVFrame *frame, AVSubtitle *sub) {
    int got_frame = 0;

    do {
        int ret = -1;

        if (d->queue->abort_request)
            return -1;

        if (!d->packet_pending || d->queue->serial != d->pkt_serial) {
            AVPacket pkt;
            do {
                if (d->queue->nb_packets == 0)
                    SDL_CondSignal(d->empty_queue_cond);
              	//从packet_queue中获取pkt
                if (packet_queue_get_or_buffering(ffp, d->queue, &pkt, &d->pkt_serial, &d->finished) < 0)
                    return -1;
                if (pkt.data == flush_pkt.data) {
                    avcodec_flush_buffers(d->avctx);
                    d->finished = 0;
                    d->next_pts = d->start_pts;
                    d->next_pts_tb = d->start_pts_tb;
                }
            } while (pkt.data == flush_pkt.data || d->queue->serial != d->pkt_serial);
            av_packet_unref(&d->pkt);
          	//将包pkt传递给解码器d
            d->pkt_temp = d->pkt = pkt;
            d->packet_pending = 1;
        }

        switch (d->avctx->codec_type) {
            case AVMEDIA_TYPE_VIDEO: {
              	//调用ffmpeg方法：avcodec_deco_video2()来解码。
                ret = avcodec_decode_video2(d->avctx, frame, &got_frame, &d->pkt_temp);
                if (got_frame) {
                    ffp->stat.vdps = SDL_SpeedSamplerAdd(&ffp->vdps_sampler, FFP_SHOW_VDPS_AVCODEC, "vdps[avcodec]");
                    if (ffp->decoder_reorder_pts == -1) {
                        frame->pts = av_frame_get_best_effort_timestamp(frame);
                    } else if (!ffp->decoder_reorder_pts) {
                        frame->pts = frame->pkt_dts;
                    }
                }
                }
                break;
            case AVMEDIA_TYPE_AUDIO:
            		//调用ffmpeg方法：avcodec_decode_audio4()来解码
                ret = avcodec_decode_audio4(d->avctx, frame, &got_frame, &d->pkt_temp);
                if (got_frame) {
                    AVRational tb = (AVRational){1, frame->sample_rate};
                    if (frame->pts != AV_NOPTS_VALUE)
                        frame->pts = av_rescale_q(frame->pts, av_codec_get_pkt_timebase(d->avctx), tb);
                    else if (d->next_pts != AV_NOPTS_VALUE)
                        frame->pts = av_rescale_q(d->next_pts, d->next_pts_tb, tb);
                    if (frame->pts != AV_NOPTS_VALUE) {
                        d->next_pts = frame->pts + frame->nb_samples;
                        d->next_pts_tb = tb;
                    }
                }
                break;
            case AVMEDIA_TYPE_SUBTITLE:
                ret = avcodec_decode_subtitle2(d->avctx, sub, &got_frame, &d->pkt_temp);
                break;
            default:
                break;
        }

        if (ret < 0) {
            d->packet_pending = 0;
        } else {
            d->pkt_temp.dts =
            d->pkt_temp.pts = AV_NOPTS_VALUE;
            if (d->pkt_temp.data) {
                if (d->avctx->codec_type != AVMEDIA_TYPE_AUDIO)
                    ret = d->pkt_temp.size;
                d->pkt_temp.data += ret;
                d->pkt_temp.size -= ret;
                if (d->pkt_temp.size <= 0)
                    d->packet_pending = 0;
            } else {
                if (!got_frame) {
                    d->packet_pending = 0;
                    d->finished = d->pkt_serial;
                }
            }
        }
    } while (!got_frame && !d->finished);

    return got_frame;
}
```



那么再看回到最新的`decoder_decode_frame()`方法中，首先解码器要从包队列`PakcetQueue`中读取出包数据，再输送到ffmpeg解码器中。那么这个读取包队列中已经缓存好的包数据的方法是：

``` c
static int packet_queue_get_or_buffering(FFPlayer *ffp, PacketQueue *q, AVPacket *pkt, int *serial, int *finished)
{
    assert(finished);
    if (!ffp->packet_buffering)
        return packet_queue_get(q, pkt, 1, serial);

    while (1) {
        int new_packet = packet_queue_get(q, pkt, 0, serial);
        if (new_packet < 0)
            return -1;
        else if (new_packet == 0) {
            //=0表示no packet，因此要再取
            if (q->is_buffer_indicator && !*finished)
                ffp_toggle_buffering(ffp, 1);
            //阻塞，直到从包队列中取出队列头的包，并填充到pkt
            new_packet = packet_queue_get(q, pkt, 1, serial);
            if (new_packet < 0)
                return -1;
        }

        if (*finished == *serial) {
            av_packet_unref(pkt);
            continue;
        }
        else
            break;
    }

    return 1;
}
```

即读取包pkt是会阻塞的，直到`3.6.4`章节介绍的视频读取线程读取并解封装包pkt，并放入`PacketQueue`，这里才能从阻塞返回并继续塞给解码器。



### 小结

1. 0.8.7开始，`decode_decode_frame()`函数借助ffmpeg的两个方法来完成解码：

   1. `int avcodec_send_packet(AVCodecContex* *avctx, const AVPacket *avpkt);`往解码器里面发送pkt数据。
   2. `int avcodec_receive_frame(AVCodecContext *avctx, AVFrame *frame);`从解码器里面读取出frame帧数据。

2. 而在0.8.7之前，音频和视频的解码都各自分别使用一个不同的解码函数：

   1. 视频：

      ``` c
      //已被废弃
      int avcodec_decode_video2(AVCodecContext *avctx, AVFrame *picture,
                               int *got_picture_ptr,
                               const AVPacket *avpkt);
      ```

   2. 音频：

      ``` c
      //已被废弃
      int avcodec_decode_audio4(AVCodecContext *avctx, AVFrame *frame,
                                int *got_frame_ptr, const AVPacket *avpkt)
      ```

3. 解码字幕的函数：

   ``` c
   int avcodec_decode_subtitle2(AVCodecContext *avctx, AVSubtitle *sub,
                               int *got_sub_ptr,
                               AVPacket *avpkt);
   ```

4. 从字幕的解码流程中可以看出解码的大致逻辑为：

   1. 循环地调用`decoder_decode_frame（）`，在这个方法里面对视频，音频和字幕3种流用switch语句来分别处理解码。当然，在音频解码`audio_thread`和视频解码`video_thread`中同样会调用这个方法的。
   2. 解码前，先从`PacketQueue`读取包数据，这个数据从哪里来？从`read_thread()`函数中调用的ffmpeg的函数：`av_read_frame(ic, pkt);`来的。
   3. 解码时，先塞给解码器pkt数据，再从解码器中读出解码好的frame数据。
   4. 再把frame数据入队`FrameQueue`，留给稍后的渲染器来从`FrameQueue`中读取 



## 3 音频解码线程`audio_thread`



``` c
static int audio_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFrame *frame = av_frame_alloc();//分配一个AVFrame
    Frame *af;//从FrameQueue sampq中取出来的，要写入数据的Frame
#if CONFIG_AVFILTER
    int last_serial = -1;
    int64_t dec_channel_layout;
    int reconfigure;
#endif
    int got_frame = 0;
    AVRational tb;//分子分母对(ffmpeg为了准确性和避免转换，定义了一个分子分母对来取代float)
    int ret = 0;
    int audio_accurate_seek_fail = 0;
    int64_t audio_seek_pos = 0;
    double frame_pts = 0;
    double audio_clock = 0;
    int64_t now = 0;
    double samples_duration = 0;
    int64_t deviation = 0;
    int64_t deviation2 = 0;
    int64_t deviation3 = 0;

    if (!frame)
        return AVERROR(ENOMEM);

    do {
        ffp_audio_statistic_l(ffp);
        //音频解码
        if ((got_frame = decoder_decode_frame(ffp, &is->auddec, frame, NULL)) < 0)
            goto the_end;
        //当解码成功
        if (got_frame) {
                tb = (AVRational){1, frame->sample_rate};
                //处理accurate_seek
                if (ffp->enable_accurate_seek && is->audio_accurate_seek_req && !is->seek_req) {
                    frame_pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
                    now = av_gettime_relative() / 1000;
                    if (!isnan(frame_pts)) {
                        samples_duration = (double) frame->nb_samples / frame->sample_rate;
                        audio_clock = frame_pts + samples_duration;
                        is->accurate_seek_aframe_pts = audio_clock * 1000 * 1000;
                        audio_seek_pos = is->seek_pos;
                        deviation = llabs((int64_t)(audio_clock * 1000 * 1000) - is->seek_pos);
                        if ((audio_clock * 1000 * 1000 < is->seek_pos ) || deviation > MAX_DEVIATION) {
                            if (is->drop_aframe_count == 0) {
                                SDL_LockMutex(is->accurate_seek_mutex);
                                if (is->accurate_seek_start_time <= 0 && (is->video_stream < 0 || is->video_accurate_seek_req)) {
                                    is->accurate_seek_start_time = now;
                                }
                                SDL_UnlockMutex(is->accurate_seek_mutex);
                                av_log(NULL, AV_LOG_INFO, "audio accurate_seek start, is->seek_pos=%lld, audio_clock=%lf, is->accurate_seek_start_time = %lld\n", is->seek_pos, audio_clock, is->accurate_seek_start_time);
                            }
                            is->drop_aframe_count++;
                            while (is->video_accurate_seek_req && !is->abort_request) {
                                int64_t vpts = is->accurate_seek_vframe_pts;
                                deviation2 = vpts  - audio_clock * 1000 * 1000;
                                deviation3 = vpts  - is->seek_pos;
                                if (deviation2 > -100 * 1000 && deviation3 < 0) {

                                    break;
                                } else {
                                    av_usleep(20 * 1000);
                                }
                                now = av_gettime_relative() / 1000;
                                if ((now - is->accurate_seek_start_time) > ffp->accurate_seek_timeout) {
                                    break;
                                }
                            }

                            if(!is->video_accurate_seek_req && is->video_stream >= 0 && audio_clock * 1000 * 1000 > is->accurate_seek_vframe_pts) {
                                audio_accurate_seek_fail = 1;
                            } else {
                                now = av_gettime_relative() / 1000;
                                if ((now - is->accurate_seek_start_time) <= ffp->accurate_seek_timeout) {
                                    av_frame_unref(frame);
                                    continue;  // drop some old frame when do accurate seek
                                } else {
                                    audio_accurate_seek_fail = 1;
                                }
                            }
                        } else {
                            if (audio_seek_pos == is->seek_pos) {
                                av_log(NULL, AV_LOG_INFO, "audio accurate_seek is ok, is->drop_aframe_count=%d, audio_clock = %lf\n", is->drop_aframe_count, audio_clock);
                                is->drop_aframe_count       = 0;
                                SDL_LockMutex(is->accurate_seek_mutex);
                                is->audio_accurate_seek_req = 0;
                                SDL_CondSignal(is->video_accurate_seek_cond);
                                if (audio_seek_pos == is->seek_pos && is->video_accurate_seek_req && !is->abort_request) {
                                    SDL_CondWaitTimeout(is->audio_accurate_seek_cond, is->accurate_seek_mutex, ffp->accurate_seek_timeout);
                                } else {
                                    ffp_notify_msg2(ffp, FFP_MSG_ACCURATE_SEEK_COMPLETE, (int)(audio_clock * 1000));
                                }

                                if (audio_seek_pos != is->seek_pos && !is->abort_request) {
                                    is->audio_accurate_seek_req = 1;
                                    SDL_UnlockMutex(is->accurate_seek_mutex);
                                    av_frame_unref(frame);
                                    continue;
                                }

                                SDL_UnlockMutex(is->accurate_seek_mutex);
                            }
                        }
                    } else {
                        audio_accurate_seek_fail = 1;
                    }
                    if (audio_accurate_seek_fail) {
                        av_log(NULL, AV_LOG_INFO, "audio accurate_seek is error, is->drop_aframe_count=%d, now = %lld, audio_clock = %lf\n", is->drop_aframe_count, now, audio_clock);
                        is->drop_aframe_count       = 0;
                        SDL_LockMutex(is->accurate_seek_mutex);
                        is->audio_accurate_seek_req = 0;
                        SDL_CondSignal(is->video_accurate_seek_cond);
                        if (is->video_accurate_seek_req && !is->abort_request) {
                            SDL_CondWaitTimeout(is->audio_accurate_seek_cond, is->accurate_seek_mutex, ffp->accurate_seek_timeout);
                        } else {
                            ffp_notify_msg2(ffp, FFP_MSG_ACCURATE_SEEK_COMPLETE, (int)(audio_clock * 1000));
                        }
                        SDL_UnlockMutex(is->accurate_seek_mutex);
                    }
                    is->accurate_seek_start_time = 0;
                    audio_accurate_seek_fail = 0;
                }

#if CONFIG_AVFILTER
                dec_channel_layout = get_valid_channel_layout(frame->channel_layout, frame->channels);

                reconfigure =
                    cmp_audio_fmts(is->audio_filter_src.fmt, is->audio_filter_src.channels,
                                   frame->format, frame->channels)    ||
                    is->audio_filter_src.channel_layout != dec_channel_layout ||
                    is->audio_filter_src.freq           != frame->sample_rate ||
                    is->auddec.pkt_serial               != last_serial        ||
                    ffp->af_changed;

                if (reconfigure) {
                    SDL_LockMutex(ffp->af_mutex);
                    ffp->af_changed = 0;
                    char buf1[1024], buf2[1024];
                    av_get_channel_layout_string(buf1, sizeof(buf1), -1, is->audio_filter_src.channel_layout);
                    av_get_channel_layout_string(buf2, sizeof(buf2), -1, dec_channel_layout);
                    av_log(NULL, AV_LOG_DEBUG,
                           "Audio frame changed from rate:%d ch:%d fmt:%s layout:%s serial:%d to rate:%d ch:%d fmt:%s layout:%s serial:%d\n",
                           is->audio_filter_src.freq, is->audio_filter_src.channels, av_get_sample_fmt_name(is->audio_filter_src.fmt), buf1, last_serial,
                           frame->sample_rate, frame->channels, av_get_sample_fmt_name(frame->format), buf2, is->auddec.pkt_serial);

                    is->audio_filter_src.fmt            = frame->format;
                    is->audio_filter_src.channels       = frame->channels;
                    is->audio_filter_src.channel_layout = dec_channel_layout;
                    is->audio_filter_src.freq           = frame->sample_rate;
                    last_serial                         = is->auddec.pkt_serial;

                    if ((ret = configure_audio_filters(ffp, ffp->afilters, 1)) < 0) {
                        SDL_UnlockMutex(ffp->af_mutex);
                        goto the_end;
                    }
                    SDL_UnlockMutex(ffp->af_mutex);
                }

            if ((ret = av_buffersrc_add_frame(is->in_audio_filter, frame)) < 0)
                goto the_end;

            while ((ret = av_buffersink_get_frame_flags(is->out_audio_filter, frame, 0)) >= 0) {
                tb = av_buffersink_get_time_base(is->out_audio_filter);
#endif
                if (!(af = frame_queue_peek_writable(&is->sampq)))//如果sampq无法写入，则失败
                    goto the_end;

                af->pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
                af->pos = frame->pkt_pos;
                af->serial = is->auddec.pkt_serial;
                af->duration = av_q2d((AVRational){frame->nb_samples, frame->sample_rate});
                //Move everything contained in src to dst and reset src.将解码出来的AVFrame传给af->frame
                av_frame_move_ref(af->frame, frame);
                //将af->frame入队
                frame_queue_push(&is->sampq);

#if CONFIG_AVFILTER
                if (is->audioq.serial != is->auddec.pkt_serial)
                    break;
            }
            if (ret == AVERROR_EOF)
                is->auddec.finished = is->auddec.pkt_serial;
#endif
        }
    } while (ret >= 0 || ret == AVERROR(EAGAIN) || ret == AVERROR_EOF);
 the_end:
#if CONFIG_AVFILTER
    avfilter_graph_free(&is->agraph);
#endif
    av_frame_free(&frame);
    return ret;
}
```



音频解码这里暂时不去分析解码之后的seek操作，所以和字幕解码没什么差别，没什么好分析的。



## 4 视频解码线程`video_thread`

终于来到视频解码了...

``` c
static int video_thread(void *arg)
{
    FFPlayer *ffp = (FFPlayer *)arg;
    int       ret = 0;
		//如果node_vdec不为null。
    if (ffp->node_vdec) {
      	//调用解码器的解码方法，进入循环
        ret = ffpipenode_run_sync(ffp->node_vdec);
    }
    return ret;
}
```

最后是走到了`IJKFF_Pipenode`的`func_run_sync()`函数中

``` c
static int ffplay_video_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFrame *frame = av_frame_alloc();//创建一个新的AVFrame
    double pts;
    double duration;
    int ret;
    AVRational tb = is->video_st->time_base;
    AVRational frame_rate = av_guess_frame_rate(is->ic, is->video_st, NULL);
    int64_t dst_pts = -1;
    int64_t last_dst_pts = -1;
    int retry_convert_image = 0;
    int convert_frame_count = 0;

#if CONFIG_AVFILTER
    AVFilterGraph *graph = avfilter_graph_alloc();
    AVFilterContext *filt_out = NULL, *filt_in = NULL;
    int last_w = 0;
    int last_h = 0;
    enum AVPixelFormat last_format = -2;
    int last_serial = -1;
    int last_vfilter_idx = 0;
    if (!graph) {
        av_frame_free(&frame);
        return AVERROR(ENOMEM);
    }

#else
    ffp_notify_msg2(ffp, FFP_MSG_VIDEO_ROTATION_CHANGED, ffp_get_video_rotate_degrees(ffp));
#endif

    if (!frame) {
#if CONFIG_AVFILTER
        avfilter_graph_free(&graph);
#endif
        return AVERROR(ENOMEM);
    }
    //开启无限循环，无限地去从packet_queue中拿取pkt来解码。
    for (;;) {
        ret = get_video_frame(ffp, frame);//解码，并将解码后的帧数据存放在frame中
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

//省略了AV_FILTER部分的代码
            duration = (frame_rate.num && frame_rate.den ? av_q2d((AVRational){frame_rate.den, frame_rate.num}) : 0);
            pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
            //将frame入队到pictq中，来让渲染线程读取。
            ret = queue_picture(ffp, frame, pts, duration, frame->pkt_pos, is->viddec.pkt_serial);
            av_frame_unref(frame);

        if (ret < 0)
            goto the_end;
    }
 the_end:
#if CONFIG_AVFILTER
    avfilter_graph_free(&graph);
#endif
    av_log(NULL, AV_LOG_INFO, "convert image convert_frame_count = %d\n", convert_frame_count);
    av_frame_free(&frame);
    return 0;
}
```



简略为：

``` c
static int ffplay_video_thread(void *arg){
  for(;;){
    ret = get_video_frame(ffp, frame);//解码，并将解码后的帧数据存放在frame中
    //将frame入队到pictq中，来让渲染线程读取。
    ret = queue_picture(ffp, frame, pts, duration, frame->pkt_pos, is->viddec.pkt_serial);
  }
}
```

那么看下解码函数：

``` c
static int get_video_frame(FFPlayer *ffp, AVFrame *frame)
{
    VideoState *is = ffp->is;
    int got_picture;

    ffp_video_statistic_l(ffp);
    //解码，并将视频帧数据填充到frame中，可能阻塞
    if ((got_picture = decoder_decode_frame(ffp, &is->viddec, frame, NULL)) < 0)
        return -1;

    if (got_picture) {
        double dpts = NAN;

        if (frame->pts != AV_NOPTS_VALUE)
            dpts = av_q2d(is->video_st->time_base) * frame->pts;

        frame->sample_aspect_ratio = av_guess_sample_aspect_ratio(is->ic, is->video_st, frame);

        if (ffp->framedrop>0 || (ffp->framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) {
            ffp->stat.decode_frame_count++;
            if (frame->pts != AV_NOPTS_VALUE) {
                double diff = dpts - get_master_clock(is);
                if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
                    diff - is->frame_last_filter_delay < 0 &&
                    is->viddec.pkt_serial == is->vidclk.serial &&
                    is->videoq.nb_packets) {
                    is->frame_drops_early++;
                    is->continuous_frame_drops_early++;
                    if (is->continuous_frame_drops_early > ffp->framedrop) {
                        is->continuous_frame_drops_early = 0;
                    } else {
                        ffp->stat.drop_frame_count++;
                        ffp->stat.drop_frame_rate = (float)(ffp->stat.drop_frame_count) / (float)(ffp->stat.decode_frame_count);
                        av_frame_unref(frame);
                        got_picture = 0;
                    }
                }
            }
        }
    }

    return got_picture;
}
```

那么这里又是用的和字幕解码、音频解码一样的解码函数：`decoder_decode_frame`，就不重复提了。

``` c
//将src_frame入队到picq中，让渲染线程渲染。
static int queue_picture(FFPlayer *ffp, AVFrame *src_frame, double pts, double duration, int64_t pos, int serial)
{
    VideoState *is = ffp->is;
    Frame *vp;
    int video_accurate_seek_fail = 0;
    int64_t video_seek_pos = 0;
    int64_t now = 0;
    int64_t deviation = 0;

    int64_t deviation2 = 0;
    int64_t deviation3 = 0;
    //处理精确seek
    if (ffp->enable_accurate_seek && is->video_accurate_seek_req && !is->seek_req) {
        if (!isnan(pts)) {
            video_seek_pos = is->seek_pos;
            is->accurate_seek_vframe_pts = pts * 1000 * 1000;
            deviation = llabs((int64_t)(pts * 1000 * 1000) - is->seek_pos);
            if ((pts * 1000 * 1000 < is->seek_pos) || deviation > MAX_DEVIATION) {
                now = av_gettime_relative() / 1000;
                if (is->drop_vframe_count == 0) {
                    SDL_LockMutex(is->accurate_seek_mutex);
                    if (is->accurate_seek_start_time <= 0 && (is->audio_stream < 0 || is->audio_accurate_seek_req)) {
                        is->accurate_seek_start_time = now;
                    }
                    SDL_UnlockMutex(is->accurate_seek_mutex);
                    av_log(NULL, AV_LOG_INFO, "video accurate_seek start, is->seek_pos=%lld, pts=%lf, is->accurate_seek_time = %lld\n", is->seek_pos, pts, is->accurate_seek_start_time);
                }
                is->drop_vframe_count++;

                while (is->audio_accurate_seek_req && !is->abort_request) {
                    int64_t apts = is->accurate_seek_aframe_pts ;
                    deviation2 = apts - pts * 1000 * 1000;
                    deviation3 = apts - is->seek_pos;

                    if (deviation2 > -100 * 1000 && deviation3 < 0) {
                        break;
                    } else {
                        av_usleep(20 * 1000);
                    }
                    now = av_gettime_relative() / 1000;
                    if ((now - is->accurate_seek_start_time) > ffp->accurate_seek_timeout) {
                        break;
                    }
                }

                if ((now - is->accurate_seek_start_time) <= ffp->accurate_seek_timeout) {
                    return 1;  // drop some old frame when do accurate seek
                } else {
                    av_log(NULL, AV_LOG_WARNING, "video accurate_seek is error, is->drop_vframe_count=%d, now = %lld, pts = %lf\n", is->drop_vframe_count, now, pts);
                    video_accurate_seek_fail = 1;  // if KEY_FRAME interval too big, disable accurate seek
                }
            } else {
                av_log(NULL, AV_LOG_INFO, "video accurate_seek is ok, is->drop_vframe_count =%d, is->seek_pos=%lld, pts=%lf\n", is->drop_vframe_count, is->seek_pos, pts);
                if (video_seek_pos == is->seek_pos) {
                    is->drop_vframe_count       = 0;
                    SDL_LockMutex(is->accurate_seek_mutex);
                    is->video_accurate_seek_req = 0;
                    SDL_CondSignal(is->audio_accurate_seek_cond);
                    if (video_seek_pos == is->seek_pos && is->audio_accurate_seek_req && !is->abort_request) {
                        SDL_CondWaitTimeout(is->video_accurate_seek_cond, is->accurate_seek_mutex, ffp->accurate_seek_timeout);
                    } else {
                        ffp_notify_msg2(ffp, FFP_MSG_ACCURATE_SEEK_COMPLETE, (int)(pts * 1000));
                    }
                    if (video_seek_pos != is->seek_pos && !is->abort_request) {
                        is->video_accurate_seek_req = 1;
                        SDL_UnlockMutex(is->accurate_seek_mutex);
                        return 1;
                    }

                    SDL_UnlockMutex(is->accurate_seek_mutex);
                }
            }
        } else {
            video_accurate_seek_fail = 1;
        }

        if (video_accurate_seek_fail) {
            is->drop_vframe_count = 0;
            SDL_LockMutex(is->accurate_seek_mutex);
            is->video_accurate_seek_req = 0;
            SDL_CondSignal(is->audio_accurate_seek_cond);
            if (is->audio_accurate_seek_req && !is->abort_request) {
                SDL_CondWaitTimeout(is->video_accurate_seek_cond, is->accurate_seek_mutex, ffp->accurate_seek_timeout);
            } else {
                if (!isnan(pts)) {
                    ffp_notify_msg2(ffp, FFP_MSG_ACCURATE_SEEK_COMPLETE, (int)(pts * 1000));
                } else {
                    ffp_notify_msg2(ffp, FFP_MSG_ACCURATE_SEEK_COMPLETE, 0);
                }
            }
            SDL_UnlockMutex(is->accurate_seek_mutex);
        }
        is->accurate_seek_start_time = 0;
        video_accurate_seek_fail = 0;
        is->accurate_seek_vframe_pts = 0;
    }

#if defined(DEBUG_SYNC)
    printf("frame_type=%c pts=%0.3f\n",
           av_get_picture_type_char(src_frame->pict_type), pts);
#endif

    if (!(vp = frame_queue_peek_writable(&is->pictq)))
        return -1;

    vp->sar = src_frame->sample_aspect_ratio;
#ifdef FFP_MERGE
    vp->uploaded = 0;
#endif

    /* alloc or resize hardware picture buffer */
    if (!vp->bmp || !vp->allocated ||
        vp->width  != src_frame->width ||
        vp->height != src_frame->height ||
        vp->format != src_frame->format) {

        if (vp->width != src_frame->width || vp->height != src_frame->height)
            ffp_notify_msg3(ffp, FFP_MSG_VIDEO_SIZE_CHANGED, src_frame->width, src_frame->height);

        vp->allocated = 0;
        vp->width = src_frame->width;
        vp->height = src_frame->height;
        vp->format = src_frame->format;

        /* the allocation must be done in the main thread to avoid
           locking problems. */
        alloc_picture(ffp, src_frame->format);

        if (is->videoq.abort_request)
            return -1;
    }

    /* if the frame is not skipped, then display it */
    if (vp->bmp) {
        /* get a pointer on the bitmap */
        SDL_VoutLockYUVOverlay(vp->bmp);//加锁

#ifdef FFP_MERGE
#if CONFIG_AVFILTER
        // FIXME use direct rendering
        av_image_copy(data, linesize, (const uint8_t **)src_frame->data, src_frame->linesize,
                        src_frame->format, vp->width, vp->height);
#else
        // sws_getCachedContext(...);
#endif
#endif
        // FIXME: set swscale options
        //将src_frame中的帧数据填充到vp->bmp中，这个vp->bmp其实指的是bitmap？
        if (SDL_VoutFillFrameYUVOverlay(vp->bmp, src_frame) < 0) {
            av_log(NULL, AV_LOG_FATAL, "Cannot initialize the conversion context\n");
            exit(1);
        }
        /* update the bitmap content */
        SDL_VoutUnlockYUVOverlay(vp->bmp);//解锁

        vp->pts = pts;
        vp->duration = duration;
        vp->pos = pos;
        vp->serial = serial;
        vp->sar = src_frame->sample_aspect_ratio;
        vp->bmp->sar_num = vp->sar.num;
        vp->bmp->sar_den = vp->sar.den;

#ifdef FFP_MERGE
        av_frame_move_ref(vp->frame, src_frame);
#endif
        frame_queue_push(&is->pictq);
        if (!is->viddec.first_frame_decoded) {
            ALOGD("Video: first frame decoded\n");
            ffp_notify_msg1(ffp, FFP_MSG_VIDEO_DECODED_START);
            is->viddec.first_frame_decoded_time = SDL_GetTickHR();
            is->viddec.first_frame_decoded = 1;
        }
    }
    return 0;
}
```

这里重点看下将frame数据填充到vp->bmp数据中的这个操作。

bmp长得非常像bitmap，看来意思是将帧数据填充到图像数据中的意思了。

``` c
int SDL_VoutFillFrameYUVOverlay(SDL_VoutOverlay *overlay, const AVFrame *frame)
{
    if (!overlay || !overlay->func_fill_frame)
        return -1;

    return overlay->func_fill_frame(overlay, frame);
}
```

``` c
static int func_fill_frame(SDL_VoutOverlay *overlay, const AVFrame *frame){
	//...
    overlay_fill(overlay, opaque->linked_frame, opaque->planes);
	//...
}

static void overlay_fill(SDL_VoutOverlay *overlay, AVFrame *frame, int planes)
{
    overlay->planes = planes;

    for (int i = 0; i < AV_NUM_DATA_POINTERS; ++i) {
        //数组的复制 
        overlay->pixels[i] = frame->data[i];
        overlay->pitches[i] = frame->linesize[i];
    }
}
```

那么到这里，应该是将`AVFrame`中的数据全部复制到这个`vp->bmp`中了，而他是：`*SDL_VoutOverlay*`



## 5 视频渲染线程



``` c
//创建视频刷新线程
is->video_refresh_tid = SDL_CreateThreadEx(&is->_video_refresh_tid, video_refresh_thread, ffp, "ff_vout");
```

创建一个线程专门用于渲染视频。在看代码之前，先了解一下视频渲染要做什么：

1. 从`FrameQueue`中拿取每一帧解码完的原始图像帧数据。
2. 将帧数据发送到显示设备，让对应设备将图像数据画出来。
3. 这是一个循环的过程，解码线程不断解码出图像帧，这边的渲染线程不断地读取图像帧并输送到渲染设备。



``` c
//	ijkmedia/ijkplayer/ff_ffplay.c
static int video_refresh_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    double remaining_time = 0.0;
  	//循环，如果没有中断请求，那么就一直尝试去渲染。
    while (!is->abort_request) {
        if (remaining_time > 0.0)
            av_usleep((int)(int64_t)(remaining_time * 1000000.0));
        remaining_time = REFRESH_RATE;
        if (is->show_mode != SHOW_MODE_NONE && (!is->paused || is->force_refresh))
          	//刷新视频
            video_refresh(ffp, &remaining_time);
    }

    return 0;
}
```

``` c
//	ijkmedia/ijkplayer/ff_ffplay.c

/* called to display each frame */
static void video_refresh(FFPlayer *opaque, double *remaining_time)
{
    FFPlayer *ffp = opaque;
    VideoState *is = ffp->is;
    double time;

    Frame *sp, *sp2;
    //处理时钟。
    if (!is->paused && get_master_sync_type(is) == AV_SYNC_EXTERNAL_CLOCK && is->realtime)
        check_external_clock_speed(is);

    if (!ffp->display_disable && is->show_mode != SHOW_MODE_VIDEO && is->audio_st) {
        time = av_gettime_relative() / 1000000.0;
        if (is->force_refresh || is->last_vis_time + ffp->rdftspeed < time) {
            //①
            video_display2(ffp);
            is->last_vis_time = time;
        }
        *remaining_time = FFMIN(*remaining_time, is->last_vis_time + ffp->rdftspeed - time);
    }

    if (is->video_st) {
retry:
        if (frame_queue_nb_remaining(&is->pictq) == 0) {
            // nothing to do, no picture to display in the queue
        } else {
            double last_duration, duration, delay;
            Frame *vp, *lastvp;

            /* dequeue the picture */
            lastvp = frame_queue_peek_last(&is->pictq);
            vp = frame_queue_peek(&is->pictq);

            if (vp->serial != is->videoq.serial) {
                frame_queue_next(&is->pictq);
                goto retry;
            }

            if (lastvp->serial != vp->serial)
                is->frame_timer = av_gettime_relative() / 1000000.0;

            if (is->paused)
                goto display;

            /* compute nominal last_duration */
            last_duration = vp_duration(is, lastvp, vp);
            delay = compute_target_delay(ffp, last_duration, is);

            time= av_gettime_relative()/1000000.0;
            if (isnan(is->frame_timer) || time < is->frame_timer)
                is->frame_timer = time;
            if (time < is->frame_timer + delay) {
                *remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
                goto display;
            }

            is->frame_timer += delay;
            if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
                is->frame_timer = time;

            SDL_LockMutex(is->pictq.mutex);
            if (!isnan(vp->pts))
                update_video_pts(is, vp->pts, vp->pos, vp->serial);
            SDL_UnlockMutex(is->pictq.mutex);

            if (frame_queue_nb_remaining(&is->pictq) > 1) {
                Frame *nextvp = frame_queue_peek_next(&is->pictq);
                duration = vp_duration(is, vp, nextvp);
                if(!is->step && (ffp->framedrop > 0 || (ffp->framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) && time > is->frame_timer + duration) {
                    frame_queue_next(&is->pictq);
                    goto retry;
                }
            }

            if (is->subtitle_st) {
                while (frame_queue_nb_remaining(&is->subpq) > 0) {
                    sp = frame_queue_peek(&is->subpq);

                    if (frame_queue_nb_remaining(&is->subpq) > 1)
                        sp2 = frame_queue_peek_next(&is->subpq);
                    else
                        sp2 = NULL;

                    if (sp->serial != is->subtitleq.serial
                            || (is->vidclk.pts > (sp->pts + ((float) sp->sub.end_display_time / 1000)))
                            || (sp2 && is->vidclk.pts > (sp2->pts + ((float) sp2->sub.start_display_time / 1000))))
                    {
                        if (sp->uploaded) {
                            ffp_notify_msg4(ffp, FFP_MSG_TIMED_TEXT, 0, 0, "", 1);
                        }
                        frame_queue_next(&is->subpq);
                    } else {
                        break;
                    }
                }
            }

            frame_queue_next(&is->pictq);
            is->force_refresh = 1;

            SDL_LockMutex(ffp->is->play_mutex);
            if (is->step) {
                is->step = 0;
                if (!is->paused)
                    stream_update_pause_l(ffp);
            }
            SDL_UnlockMutex(ffp->is->play_mutex);
        }
display:
        /* display picture */
        if (!ffp->display_disable && is->force_refresh && is->show_mode == SHOW_MODE_VIDEO && is->pictq.rindex_shown)
            //①
            video_display2(ffp);
    }
    is->force_refresh = 0;
    if (ffp->show_status) {
        static int64_t last_time;
        int64_t cur_time;
        int aqsize, vqsize, sqsize __unused;
        double av_diff;

        cur_time = av_gettime_relative();
        if (!last_time || (cur_time - last_time) >= 30000) {
            aqsize = 0;
            vqsize = 0;
            sqsize = 0;
            if (is->audio_st)
                aqsize = is->audioq.size;
            if (is->video_st)
                vqsize = is->videoq.size;
#ifdef FFP_MERGE
            if (is->subtitle_st)
                sqsize = is->subtitleq.size;
#else
            sqsize = 0;
#endif
            av_diff = 0;
            if (is->audio_st && is->video_st)
                av_diff = get_clock(&is->audclk) - get_clock(&is->vidclk);
            else if (is->video_st)
                av_diff = get_master_clock(is) - get_clock(&is->vidclk);
            else if (is->audio_st)
                av_diff = get_master_clock(is) - get_clock(&is->audclk);
            av_log(NULL, AV_LOG_INFO,
                   "%7.2f %s:%7.3f fd=%4d aq=%5dKB vq=%5dKB sq=%5dB f=%"PRId64"/%"PRId64"   \r",
                   get_master_clock(is),
                   (is->audio_st && is->video_st) ? "A-V" : (is->video_st ? "M-V" : (is->audio_st ? "M-A" : "   ")),
                   av_diff,
                   is->frame_drops_early + is->frame_drops_late,
                   aqsize / 1024,
                   vqsize / 1024,
                   sqsize,
                   is->video_st ? is->viddec.avctx->pts_correction_num_faulty_dts : 0,
                   is->video_st ? is->viddec.avctx->pts_correction_num_faulty_pts : 0);
            fflush(stdout);
            last_time = cur_time;
        }
    }
}
```

一长串代码，貌似有一些根据时钟来同步音视频的代码？暂时不做分析，这里面要跳转到方法（用①做了标记，有两处）：

``` c
//①
video_display2(ffp);
```

``` c
/* display the current picture, if any */
static void video_display2(FFPlayer *ffp)
{
    VideoState *is = ffp->is;
    if (is->video_st)
        video_image_display2(ffp);
}
```

``` c
static void video_image_display2(FFPlayer *ffp)
{
    VideoState *is = ffp->is;
    Frame *vp;
    Frame *sp = NULL;
    //is->pictq就是picture queue的意思。读取队列中最后一帧。
    vp = frame_queue_peek_last(&is->pictq);
    //如果帧中的SDL_VoutOverlay数据不为null，那么就开始渲染
    if (vp->bmp) {
        //如果字幕流不为空，去渲染字幕
        if (is->subtitle_st) {
            if (frame_queue_nb_remaining(&is->subpq) > 0) {
                sp = frame_queue_peek(&is->subpq);

                if (vp->pts >= sp->pts + ((float) sp->sub.start_display_time / 1000)) {
                    if (!sp->uploaded) {
                        if (sp->sub.num_rects > 0) {
                            char buffered_text[4096];
                            if (sp->sub.rects[0]->text) {
                                strncpy(buffered_text, sp->sub.rects[0]->text, 4096);
                            }
                            else if (sp->sub.rects[0]->ass) {
                                parse_ass_subtitle(sp->sub.rects[0]->ass, buffered_text);
                            }
                            ffp_notify_msg4(ffp, FFP_MSG_TIMED_TEXT, 0, 0, buffered_text, sizeof(buffered_text));
                        }
                        sp->uploaded = 1;
                    }
                }
            }
        }
        if (ffp->render_wait_start && !ffp->start_on_prepared && is->pause_req) {
            if (!ffp->first_video_frame_rendered) {
                ffp->first_video_frame_rendered = 1;
                ffp_notify_msg1(ffp, FFP_MSG_VIDEO_RENDERING_START);
            }
            while (is->pause_req && !is->abort_request) {
                SDL_Delay(20);
            }
        }
        //显示YUV数据。
        SDL_VoutDisplayYUVOverlay(ffp->vout, vp->bmp);
        ffp->stat.vfps = SDL_SpeedSamplerAdd(&ffp->vfps_sampler, FFP_SHOW_VFPS_FFPLAY, "vfps[ffplay]");
        if (!ffp->first_video_frame_rendered) {
            ffp->first_video_frame_rendered = 1;
            ffp_notify_msg1(ffp, FFP_MSG_VIDEO_RENDERING_START);
        }

        if (is->latest_video_seek_load_serial == vp->serial) {
            int latest_video_seek_load_serial = __atomic_exchange_n(&(is->latest_video_seek_load_serial), -1, memory_order_seq_cst);
            if (latest_video_seek_load_serial == vp->serial) {
                ffp->stat.latest_seek_load_duration = (av_gettime() - is->latest_seek_load_start_at) / 1000;
                if (ffp->av_sync_type == AV_SYNC_VIDEO_MASTER) {
                    ffp_notify_msg2(ffp, FFP_MSG_VIDEO_SEEK_RENDERING_START, 1);
                } else {
                    ffp_notify_msg2(ffp, FFP_MSG_VIDEO_SEEK_RENDERING_START, 0);
                }
            }
        }
    }
}
```

首先看到取视频帧的这一段代码：

``` c
Frame *vp;
Frame *sp = NULL;
//is->pictq就是picture queue的意思。读取队列中最后一帧。
vp = frame_queue_peek_last(&is->pictq);
```

在这里，`Frame`就是每一帧解码后的图像数据，是直接拿去显示的。而`is->pictq`就是`VideoState`里面的解码后的图像队列`FrameQueue`。

看一下`Frame`和`FrameQueue`

``` c
typedef struct Frame {
    AVFrame *frame;//ffmpeg定义的数据结构，里面存着buffer，存着真实的yuv图像数据
    AVSubtitle sub;//字幕数据
    int serial;
    double pts;           /* presentation timestamp for the frame */
    double duration;      /* estimated duration of the frame */
    int64_t pos;          /* byte position of the frame in the input file */
#ifdef FFP_MERGE
    SDL_Texture *bmp;
#else
    SDL_VoutOverlay *bmp;//vout设备
#endif
    int allocated;
    int width;
    int height;
    int format;
    AVRational sar;
    int uploaded;
} Frame;

typedef struct FrameQueue {
    Frame queue[FRAME_QUEUE_SIZE];//数组
    int rindex;//read index。下一个读取的下标
    int windex;//write index。下一个写入的下标
    int size;
    int max_size;
    int keep_last;
    int rindex_shown;
    SDL_mutex *mutex;
    SDL_cond *cond;
    PacketQueue *pktq;//引用的未解码的包队列
} FrameQueue;
```



然后是看到渲染的这一句：

``` c
//显示YUV数据。这个vp是Frame，而这个bmp是bitmap的意思
SDL_VoutDisplayYUVOverlay(ffp->vout, vp->bmp);
```

这里的意思是将vp->bmp中的数据输送到ffp->vout中。
