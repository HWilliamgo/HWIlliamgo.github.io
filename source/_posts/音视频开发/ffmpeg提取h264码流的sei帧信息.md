---
title: ffmpeg提取h264码流的sei帧信息
tags:
  - ffmpeg
  - 音视频
index_img: 'https://s1.ax1x.com/2020/04/04/G0w37j.md.jpg'
banner_img: 'https://s1.ax1x.com/2020/04/04/G0w37j.jpg'
categories:
  - 音视频开发
date: 2020-03-30 20:46:43
---

码流中的sei帧信息了解建议文章：

[FFmpeg代码导读系列(二)----SEI的那些事](https://www.jianshu.com/p/4d9120dfcd69)





从ffmpeg中提取h264码流中的sei信息的方式分为两种：

1. 不需要修改ffmpeg源码
2. 需要修改ffmpeg源码

本文对这两种方式提取sei信息的方式做一个总结。



### 1 不修改ffmpeg源码

#### 1 借助av_read_frame

这是侵入性最小，最简单的方式。

![GuRmpn.png](https://s1.ax1x.com/2020/03/30/GuRmpn.png)



如这张图，这是将视频提取出h264码流后，用16进制文本查看器看到的内容。

其中00,00,01,06 就表示当前遇到了sei数据。（这不是标准的seiType=5的数据，从后面的的64看出，这是seiType=100的数据，因此制定他的厂商并不是完全按照标准来的）

由于恰好地，00,00,01,06这种，由两位16进制组成的数字，刚好是16x16=256，8bit也能表示256种数据，即2^8=256。而8bit=1byte。即每两个16位的数字可以用一个byte表示。

而刚好，av_read_frame()方法提取出来的码流数据，就是字节数组。那么将提取出的AVPacket中的data来遍历，找到匹配sei规则的数据，再去解析，也是可以的了。例如：

``` c
ret = av_read_frame(ic, pkt);
if (ret == 0 && pkt != NULL && pkt->data != NULL && pkt->size > 0 && (pkt->flags & AV_PKT_FLAG_KEY)) {
    for (int k = 0; k < pkt->size; k++) {
        uint8_t nalType = pkt->data[k + 4];
        uint8_t seiType = pkt->data[k + 5];
        if (pkt->data[k] == 0 && pkt->data[k + 1] == 0 && pkt->data[k + 2] == 1 && nalType == 6 ) {
            if(seiType==要解析的seiType)
            //...
        }
    }
}
```

这种方法是一种比较简单的，没有侵入性的方法。但是效率会低一点，因为最坏的情况是要遍历整个AVPacket中的数据才能找到sei数据。



### 2 修改ffmpeg源码

ffmpeg源码中本身是对sei帧信息有做处理的。我们需要找到ffmpeg源码中对sei帧做处理的位置。

``` c
// h264_parser.c
/**
 * Parse NAL units of found picture and decode some basic information.
 *
 * @param s parser context.
 * @param avctx codec context.
 * @param buf buffer with field/frame data.
 * @param buf_size size of the buffer.
 */
static inline int parse_nal_units(AVCodecParserContext *s,
                                  AVCodecContext *avctx,
                                  const uint8_t * const buf, int buf_size)
{
    H264ParseContext *p = s->priv_data;
    H2645RBSP rbsp = { NULL };
    H2645NAL nal = { NULL };
    int buf_index, next_avc;
    unsigned int pps_id;
    unsigned int slice_type;
    int state = -1, got_reset = 0;
    int q264 = buf_size >=4 && !memcmp("Q264", buf, 4);
    int field_poc[2];
    int ret;

    /* set some sane default values */
    s->pict_type         = AV_PICTURE_TYPE_I;
    s->key_frame         = 0;
    s->picture_structure = AV_PICTURE_STRUCTURE_UNKNOWN;

    ff_h264_sei_uninit(&p->sei);
    p->sei.frame_packing.arrangement_cancel_flag = -1;

    if (!buf_size)
        return 0;

    av_fast_padded_malloc(&rbsp.rbsp_buffer, &rbsp.rbsp_buffer_alloc_size, buf_size);
    if (!rbsp.rbsp_buffer)
        return AVERROR(ENOMEM);

    buf_index     = 0;
    next_avc      = p->is_avc ? 0 : buf_size;
    for (;;) {
        const SPS *sps;
        int src_length, consumed, nalsize = 0;

        if (buf_index >= next_avc) {
            nalsize = get_nalsize(p->nal_length_size, buf, buf_size, &buf_index, avctx);
            if (nalsize < 0)
                break;
            next_avc = buf_index + nalsize;
        } else {
            buf_index = find_start_code(buf, buf_size, buf_index, next_avc);
            if (buf_index >= buf_size)
                break;
            if (buf_index >= next_avc)
                continue;
        }
        src_length = next_avc - buf_index;

        state = buf[buf_index];
        switch (state & 0x1f) {
        case H264_NAL_SLICE:
        case H264_NAL_IDR_SLICE:
            // Do not walk the whole buffer just to decode slice header
            if ((state & 0x1f) == H264_NAL_IDR_SLICE || ((state >> 5) & 0x3) == 0) {
                /* IDR or disposable slice
                 * No need to decode many bytes because MMCOs shall not be present. */
                if (src_length > 60)
                    src_length = 60;
            } else {
                /* To decode up to MMCOs */
                if (src_length > 1000)
                    src_length = 1000;
            }
            break;
        }
        consumed = ff_h2645_extract_rbsp(buf + buf_index, src_length, &rbsp, &nal, 1);
        if (consumed < 0)
            break;

        buf_index += consumed;

        ret = init_get_bits8(&nal.gb, nal.data, nal.size);
        if (ret < 0)
            goto fail;
        get_bits1(&nal.gb);
        nal.ref_idc = get_bits(&nal.gb, 2);
        nal.type    = get_bits(&nal.gb, 5);

        switch (nal.type) {
        case H264_NAL_SPS:
            ff_h264_decode_seq_parameter_set(&nal.gb, avctx, &p->ps, 0);
            break;
        case H264_NAL_PPS:
            ff_h264_decode_picture_parameter_set(&nal.gb, avctx, &p->ps,
                                                 nal.size_bits);
            break;
        case H264_NAL_SEI:
            ff_h264_sei_decode(&p->sei, &nal.gb, &p->ps, avctx);
            break;
        }
        //...
    }
    //...
}
```

以上保留了nal unit type 为 6，7，8的解码调用方法，分别是类型：SEI, PPS, SPS

其他的类型定义在 h264.h中：

``` c
/* NAL unit types */
enum {
    H264_NAL_SLICE           = 1,
    H264_NAL_DPA             = 2,
    H264_NAL_DPB             = 3,
    H264_NAL_DPC             = 4,
    H264_NAL_IDR_SLICE       = 5,
    H264_NAL_SEI             = 6,
    H264_NAL_SPS             = 7,
    H264_NAL_PPS             = 8,
    H264_NAL_AUD             = 9,
    H264_NAL_END_SEQUENCE    = 10,
    H264_NAL_END_STREAM      = 11,
    H264_NAL_FILLER_DATA     = 12,
    H264_NAL_SPS_EXT         = 13,
    H264_NAL_AUXILIARY_SLICE = 19,
};
```



回到代码的调用流程是：

`parse_nal_units` ->  `ff_h264_sei_decode`

``` c
int ff_h264_sei_decode(H264SEIContext *h, GetBitContext *gb,
                       const H264ParamSets *ps, void *logctx)
{
    int master_ret = 0;

    while (get_bits_left(gb) > 16 && show_bits(gb, 16)) {
        int type = 0;
        unsigned size = 0;
        unsigned next;
        int ret  = 0;

        do {
            if (get_bits_left(gb) < 8)
                return AVERROR_INVALIDDATA;
            type += show_bits(gb, 8);
        } while (get_bits(gb, 8) == 255);

        do {
            if (get_bits_left(gb) < 8)
                return AVERROR_INVALIDDATA;
            size += show_bits(gb, 8);
        } while (get_bits(gb, 8) == 255);

        if (size > get_bits_left(gb) / 8) {
            av_log(logctx, AV_LOG_ERROR, "SEI type %d size %d truncated at %d\n",
                   type, 8*size, get_bits_left(gb));
            return AVERROR_INVALIDDATA;
        }
        next = get_bits_count(gb) + 8 * size;

        h->type = type;
        switch (type) {
		// 解析SEI TYPE = 5                 
        case H264_SEI_TYPE_USER_DATA_UNREGISTERED:
            ret = decode_unregistered_user_data(&h->unregistered, gb, logctx, size);
            break;
        default:
            av_log(logctx, AV_LOG_DEBUG, "unknown SEI type %d\n", type);
        }
        if (ret < 0 && ret != AVERROR_PS_NOT_FOUND)
            return ret;
        if (ret < 0)
            master_ret = ret;

        skip_bits_long(gb, next - get_bits_count(gb));

        // FIXME check bits here
        align_get_bits(gb);
    }

    return master_ret;
}
```

这里我只列出了一个SEI TYPE，其余的类型在：h264_sei.h中

``` c
/**
 * SEI message types
 */
typedef enum {
    H264_SEI_TYPE_BUFFERING_PERIOD       = 0,   ///< buffering period (H.264, D.1.1)
    H264_SEI_TYPE_PIC_TIMING             = 1,   ///< picture timing
    H264_SEI_TYPE_FILLER_PAYLOAD         = 3,   ///< filler data
    H264_SEI_TYPE_USER_DATA_REGISTERED   = 4,   ///< registered user data as specified by Rec. ITU-T T.35
    H264_SEI_TYPE_USER_DATA_UNREGISTERED = 5,   ///< unregistered user data
    H264_SEI_TYPE_RECOVERY_POINT         = 6,   ///< recovery point (frame # to decoder sync)
    H264_SEI_TYPE_FRAME_PACKING          = 45,  ///< frame packing arrangement
    H264_SEI_TYPE_DISPLAY_ORIENTATION    = 47,  ///< display orientation
    H264_SEI_TYPE_GREEN_METADATA         = 56,  ///< GreenMPEG information
    H264_SEI_TYPE_ALTERNATIVE_TRANSFER   = 147, ///< alternative transfer
} H264_SEI_Type;
```

一般来说，seiType=5的这个类型就是留给用户自定义的类型，因为其他类型都被h264规范中的其他信息占用了，而seiType=5是他留给用户的未注册的类型，因此叫user_data_unregistered。

我们继续回到这个方法：

``` c
static int decode_unregistered_user_data(H264SEIUnregistered *h, GetBitContext *gb,
                                         void *logctx, int size)
{   
    //字符数组，也就是字符串
    uint8_t *user_data;
    int e, build, i;
    //这个16是什么意思?是uuid，因为seiType=5的时候，数据中会包含16位的uuid。这里做了个判断
    if (size < 16 || size >= INT_MAX - 16)
        return AVERROR_INVALIDDATA;
    //分配一个内存大小，规则：uuid大小+数据长度+字符串末尾空字符0
    user_data = av_malloc(16 + size + 1);
    if (!user_data)
        return AVERROR(ENOMEM);
    //i<size+16，是因为要把uuid和用户数据全部读取到这个user_data数组中
    for (i = 0; i < size + 16; i++)
        //get_bits(gb,8)的意思是一次读取8bit数据并赋值给数组。为什么是8bit？因为字符串是char[] ，而char占8位，因此一次读取8位。而2^8=256=16x16。刚好一个字节可以保存一两个16进制的数据。
        user_data[i] = get_bits(gb, 8);
    //设置末尾空字符
    user_data[i] = 0;
    e = sscanf(user_data + 16, "x264 - core %d", &build);
    if (e == 1 && build > 0)
        h->x264_build = build;
    if (e == 1 && build == 1 && !strncmp(user_data+16, "x264 - core 0000", 16))
        h->x264_build = 67;
    //释放手动分配的数组
    av_free(user_data);
    return 0;
}
```

方法中带了注释，看着还是比较简单了。我们注意到这里解析到的user_data数组，并没有对uuid和sei content做任何的处理，就直接释放掉了。那么我们自己在这里加一个处理就好了，我们的方案是：

1. 将这个user_data数组中的数据取出来。
2. 借助回调的方式上抛到外层。
3. 外层使用。



那么一步一步来。首先将数据取出来，很简单，我们把user_data中的数据再copy一份就好了，是否保留uuid看业务需要，content肯定是要取出来的。取出来之后要怎么上抛出去呢？注意看这里有一个结构体`H264SEIUnregistered`，我们往他那边保存这个字符串，在外面就可以取出来了。

``` c
typedef struct H264SEIUnregistered {
    int x264_build;
    //我们增加的，用来保存sei数据的数组。
    uint8_t *user_data;
} H264SEIUnregistered;
```

然后这样：

``` c
static int decode_unregistered_user_data(H264SEIUnregistered *h, GetBitContext *gb,
                                         void *logctx, int size)
{   
    //字符数组，也就是字符串
    uint8_t *user_data;
    int e, build, i;
    //这个16是什么意思?是uuid，因为seiType=5的时候，数据中会包含16位的uuid。这里做了个判断
    if (size < 16 || size >= INT_MAX - 16)
        return AVERROR_INVALIDDATA;
    //分配一个内存大小，规则：uuid大小+数据长度+字符串末尾空字符0
    user_data = av_malloc(16 + size + 1);
    if (!user_data)
        return AVERROR(ENOMEM);
    //i<size+16，是因为要把uuid和用户数据全部读取到这个user_data数组中
    for (i = 0; i < size + 16; i++)
        user_data[i] = get_bits(gb, 8);
    //设置末尾空字符
    user_data[i] = 0;
    e = sscanf(user_data + 16, "x264 - core %d", &build);
    if (e == 1 && build > 0)
        h->x264_build = build;
    if (e == 1 && build == 1 && !strncmp(user_data+16, "x264 - core 0000", 16))
        h->x264_build = 67;
    //从数组开始，跳过uuid的16bit的长度后，字符串长度大于0，则提取数据
    if (strlen(user_data + 16) > 0){
        //打印出来
        av_log(logctx, AV_LOG_DEBUG, "user data:\"%s\"\n", user_data + 16);

        //提取数据，+1是为了字符串末尾空字符0
        int ret = av_reallocp(&h->user_data, size + 1);
        if (ret >= 0){
            //跳过uuid
            strcpy(h->user_data, user_data + 16);
            //末尾置0
            h->user_data[size - 16] = 0;
        }
    }
    //释放手动分配的数组
    av_free(user_data);
    return 0;
}
```

然后H264SEIContext中加一个seiType的变量。外部上抛的时候要把seiType一起抛出去。

``` c
// h264_sei.h
typedef struct H264SEIContext {
    //sei类型
    int type;
    H264SEIPictureTiming picture_timing;
    H264SEIAFD afd;
    H264SEIA53Caption a53_caption;
    H264SEIUnregistered unregistered;
    H264SEIRecoveryPoint recovery_point;
    H264SEIBufferingPeriod buffering_period;
    H264SEIFramePacking frame_packing;
    H264SEIDisplayOrientation display_orientation;
    H264SEIGreenMetaData green_metadata;
    H264SEIAlternativeTransfer alternative_transfer;
} H264SEIContext;
```

而这个seiType在这里赋值：

``` c
// h264_parser.c
int ff_h264_sei_decode(H264SEIContext *h, GetBitContext *gb,
                       const H264ParamSets *ps, void *logctx)
{
    int master_ret = 0;

    while (get_bits_left(gb) > 16 && show_bits(gb, 16)) {
        int type = 0;
        unsigned size = 0;
        unsigned next;
        int ret  = 0;

        do {
            if (get_bits_left(gb) < 8)
                return AVERROR_INVALIDDATA;
            type += show_bits(gb, 8);
        } while (get_bits(gb, 8) == 255);

        do {
            if (get_bits_left(gb) < 8)
                return AVERROR_INVALIDDATA;
            size += show_bits(gb, 8);
        } while (get_bits(gb, 8) == 255);

        if (size > get_bits_left(gb) / 8) {
            av_log(logctx, AV_LOG_ERROR, "SEI type %d size %d truncated at %d\n",
                   type, 8*size, get_bits_left(gb));
            return AVERROR_INVALIDDATA;
        }
        next = get_bits_count(gb) + 8 * size;
		//赋值type
        h->type = type;
        switch (type) {
        case H264_SEI_TYPE_PIC_TIMING: // Picture timing SEI
            ret = decode_picture_timing(&h->picture_timing, gb, ps, logctx);
            break;
```

那么在解析处上抛：

``` c
// h264_parser.c
static inline int parse_nal_units(AVCodecParserContext *s,
                                  AVCodecContext *avctx,
                                  const uint8_t * const buf, int buf_size)
{
    //...
    for (;;) {
		//...
        case H264_NAL_SEI:
        	//解析数据
            ff_h264_sei_decode(&p->sei, &nal.gb, &p->ps, avctx);
        	//上抛数据
            if (p->sei.type == H264_SEI_TYPE_USER_DATA_UNREGISTERED && p->sei.unregistered.user_data != NULL){
                av_post_h264_sei(H264_SEI_TYPE_USER_DATA_UNREGISTERED, p->sei.unregistered.user_data);
            }
            break;
    }
}
```

这个`post_h264_sei(int seiType,const char* user_data)`方法就是我们定义的，用来上抛数据的。

定义位置：avcodec.h

``` c
//内部使用，用来上抛数据
void av_post_h264_sei(int seiType, const char* user_data);

//外部使用，用来设置回调
void av_set_sei_data_listener(void (*listener)(int, const char*));

```

实现位置:

``` c
//保存外部设置进来的监听器
static void (*av_h264_sei_listener)(int, const char*) = NULL;

void av_set_sei_data_listener(void (*listener)(int, const char*))
{
    av_h264_sei_listener = listener;
}

void av_post_h264_sei(int seiType, const char* user_data)
{
    void (*av_h264_sei_listener)(int, const char*) = av_h264_sei_listener;
    if (av_h264_sei_listener)
        av_h264_sei_listener(seiType, user_data);
}

```



上面是解析标准的seiType=5的场景。

有的视频厂商也不用seiType=5，他们会自己另设置一种seiType，并且解析的规范可能也与seiType=5的情况不同，例如数据中完全不规定uuid，只有内容。当然这要按照视频厂商的协议来开发了，不过总体的思路是类似的，在上述的函数对应的位置，进行对应的解析即可。



