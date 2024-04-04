# OutputStream数据结构分析—ffmpeg.c源码分析

<div id="meta-description---">struct OutputStream 是单个输出流的管理器。是由 new_video_stream() 或者 new_audio_stream() 函数申请内存，以及赋值 OutputStream 的各个字段的。而 output_streams 数组是一个全局变量，包含了所有输出文件里面的所有输出流。</div>

`struct OutputStream` 是**单个输出流的管理器**。是由 `new_video_stream()` 或者 `new_audio_stream()` 函数申请内存，以及赋值 `OutputStream` 的各个字段的。

而 `output_streams` 数组是一个全局变量，包含了所有输出文件里面的所有输出流。

```
extern OutputStream **output_streams;
extern int         nb_output_streams;
```

你在二次开发 `ffmpeg.exe` 的时候，可以用 `output_streams` 全局变量来获取到所有的**输出流**。

`struct OutputStream`数据结构 的定义如下：

```
typedef struct OutputStream {
    int file_index;          /* file index */
    int index;               /* stream index in the output file */
    int source_index;        /* InputStream index */
    AVStream *st;            /* stream in the output file */
    int encoding_needed;     /* true if encoding needed for this stream */
    int frame_number;
    /* input pts and corresponding output pts
       for A/V sync */
    struct InputStream *sync_ist; /* input stream to sync against */
    int64_t sync_opts;       /* output frame counter, could be changed to some true timestamp */ // FIXME look at frame_number
    /* pts of the first frame encoded for this stream, used for limiting
     * recording time */
    int64_t first_pts;
    /* dts of the last packet sent to the muxer */
    int64_t last_mux_dts;
    // the timebase of the packets sent to the muxer
    AVRational mux_timebase;
    AVRational enc_timebase;

    AVBSFContext            *bsf_ctx;

    AVCodecContext *enc_ctx;
    AVCodecParameters *ref_par; /* associated input codec parameters with encoders options applied */
    const AVCodec *enc;
    int64_t max_frames;
    AVFrame *filtered_frame;
    AVFrame *last_frame;
    AVPacket *pkt;
    int last_dropped;
    int last_nb0_frames[3];

    void  *hwaccel_ctx;

    /* video only */
    AVRational frame_rate;
    AVRational max_frame_rate;
    int is_cfr;
    int force_fps;
    int top_field_first;
    int rotate_overridden;
    int autoscale;
    double rotate_override_value;

    AVRational frame_aspect_ratio;

    /* forced key frames */
    int64_t forced_kf_ref_pts;
    int64_t *forced_kf_pts;
    int forced_kf_count;
    int forced_kf_index;
    char *forced_keyframes;
    AVExpr *forced_keyframes_pexpr;
    double forced_keyframes_expr_const_values[FKF_NB];

    /* audio only */
    int *audio_channels_map;             /* list of the channels id to pick from the source stream */
    int audio_channels_mapped;           /* number of channels in audio_channels_map */

    char *logfile_prefix;
    FILE *logfile;

    OutputFilter *filter;
    char *avfilter;
    char *filters;         ///< filtergraph associated to the -filter option
    char *filters_script;  ///< filtergraph script associated to the -filter_script option

    AVDictionary *encoder_opts;
    AVDictionary *sws_dict;
    AVDictionary *swr_opts;
    AVDictionary *resample_opts;
    char *apad;
    OSTFinished finished;        /* no more packets should be written for this stream */
    int unavailable;                     /* true if the steram is unavailable (possibly temporarily) */
    int stream_copy;

    // init_output_stream() has been called for this stream
    // The encoder and the bitstream filters have been initialized and the stream
    // parameters are set in the AVStream.
    int initialized;

    int inputs_done;

    const char *attachment_filename;
    int copy_initial_nonkeyframes;
    int copy_prior_start;
    char *disposition;

    int keep_pix_fmt;

    /* stats */
    // combined size of all the packets written
    uint64_t data_size;
    // number of packets send to the muxer
    uint64_t packets_written;
    // number of frames/samples sent to the encoder
    uint64_t frames_encoded;
    uint64_t samples_encoded;

    /* packet quality factor */
    int quality;

    int max_muxing_queue_size;

    /* the packets are buffered here until the muxer is ready to be initialized */
    AVFifoBuffer *muxing_queue;

    /*
     * The size of the AVPackets' buffers in queue.
     * Updated when a packet is either pushed or pulled from the queue.
     */
    size_t muxing_queue_data_size;

    /* Threshold after which max_muxing_queue_size will be in effect */
    size_t muxing_queue_data_threshold;

    /* packet picture type */
    int pict_type;

    /* frame encode sum of squared error values */
    int64_t error[4];
} OutputStream;

```

各个字段的解析如下：

**1，**`int file_index`，全局变量 output_files 数组里面的索引值。

**2，**`int index`，`oc->nb_streams - 1`，索引是容器里面最大的那个流的索引。。

**3，**`int source_index`，此输出流对应的输入流，-1 代表没有对应的输入流，通常是因为使用了滤镜，把多个输入流混合成一个输出流。`source_index` 是 `input_streams` 数组里面的索引值。

**4，**`AVStream *st`，从容器上下文里面复制过来的。

**5，**`int encoding_needed`，此输出流的数据需不需要进行编码操作。只要命令行参数没使用 copy 选项，都是需要进行编码操作的。

**6，**`int frame_number`，统计一共往编码器输入了多少个 `AVFrame`。

**7，**`struct InputStream *sync_ist`，我也不知道这个字段干嘛的，但是好像可以通过 `map` 语法改变这个字段的值。

**8，**`int64_t sync_opts`，这个字段的全称是 `sync_out_pts`，输入给编码器的每一个 AVFrame 的 pts 都是用 sync_opts 来赋值的，如下：

![1-2](struct_outputstream\1-2.png)

`sync_opts` 字段的初始值是 0，每输入一个 frame 给编码器，`sync_opts` 就会 +1。sync_opts 具体的应用代码请阅读《FFmpeg的-r帧率变换分析》

```
int64_t sync_opts; /* output frame counter, could be changed to some true timestamp */ // FIXME look at frame_number
```

不过 sync_opts 跟 frame_number 字段好像是一样的，从他的注释可以看到。

**9，**`int64_t first_pts`，此流的第一个 AVFrame 的 pts，可能是音频的pts，也可能是视频的pts。主要是用来限制输出时长的，配合那个 recording time 字段使用。

**10，**`int64_t last_mux_dts`，最近一次写入容器文件的 AVPacket 的 dts。

**11，**`AVRational mux_timebase`，复用器的时间基，也就是 AVPacket 写入文件的时候，它的 pts 需要换成 mux_timebase 的时间基。

**12，**`AVRational enc_timebase`，编码器的时间基，AVFrame 发送给编码器之前，AVFrame 的 pts 需要换成 enc_timebase的时间基。

**13，**`AVBSFContext *bsf_ctx`，字节流滤镜实例，可以用命令行参数 `-bsf` 指定，例如 `-bsf:vh264_mp4toannexb` 这个参数的作用是将MP4中的H. 264 数据转换为H. 264AnnexB 标准的编码， AnnexB标准的编码常见于实时传输流中。

**14，**`AVCodecContext *enc_ctx`，编码器实例，编码器上下文。

**15，**`AVCodecParameters *ref_par`，编码器参数，主要用于 stream copy 的场景，其实是直接复制的输入流的编码参数。

**16，**`const AVCodec *enc`，编码器信息。

**17，**`int64_t max_frames`，命令行选项 `-frames` 的值，限制此输出流最多输出多少个 AVFrame。

**18，**`AVFrame *filtered_frame`，从 FilterGraph 出口滤镜里面读出来的 AVFrame。

**19，**`AVFrame *last_frame`，指向上一次发送给 编码器的 AVFrame

**20，**`AVPacket *pkt`，从编码器里面读出来的 AVPacket。

**21，**`int last_dropped`，还是用于 帧率变换的逻辑，应该是代表最近的一个 AVFrame 是否丢弃了。

**22，**`int last_nb0_frames[3]`，还是用于 帧率变换的逻辑，不清楚是干什么的，后面补充。

**23，**`void *hwaccel_ctx`，硬件编码上下文。

---

下面是一些**视频相关**的字段。

**24，**`AVRational frame_rate`，帧率，通过 `-r` 选项指定的。

**24，**`AVRational max_frame_rate`，最大帧率，通过 `-fpsmax` 选项指定的。

**25，**`int is_cfr`，此输出流是否是常量帧率。

**26，**`int force_fps`，强制指定帧率，通过 `-force_fps` 选项指定的，是个布尔值。、

**27，**`int top_field_first`，隔行扫描视频用的。

**28，**`int rotate_overridden`，不知道干什么的，后面补充。

**29，**` int autoscale`，命令行选项 `-autoscale` 的值，在最后自动插入 `autoscale` 滤镜。

**30，**`double rotate_override_value`，不知道干什么的，后面补充。

**31，**`AVRational frame_aspect_ratio`，这是一个比例，显示播放的时候用这个比例拉伸 `width` 跟`height` 作为显示窗口，图像播放就不会扭曲了。推荐阅读，[ffmpeg解析出的视频参数PAR，DAR，SAR的意义](https://blog.csdn.net/zhizhuodewo6/article/details/105247557) 跟 [theory-videoaspectratios](https://www.animemusicvideos.org/guides/avtech3/theory-videoaspectratios.html)

---

下面是一些用于 视频强制关键帧 功能的字段，对于音频而言，每一帧都是关键帧，只有视频才有 B 帧与 P帧。

```
/* forced key frames */
int64_t forced_kf_ref_pts;
int64_t *forced_kf_pts;
int forced_kf_count;
int forced_kf_index;
char *forced_keyframes;
AVExpr *forced_keyframes_pexpr;
double forced_keyframes_expr_const_values[FKF_NB];
```

关键帧 功能的字段后面补充，我也不太清楚。

---

**32，**`int *audio_channels_map`，`int audio_channels_mapped`，给命令行的 map 语法用的。

**33，**`char *logfile_prefix`，日志前缀，不太清楚作用，后面补充。

**34，**`FILE *logfile`，不太清楚作用，后面补充，感觉跟编码器有关。

**35，**`OutputFilter *filter`，输出流关联的出口滤镜，输出流需要从这个出口滤镜读取 AVFrame。

**36，**`char *avfilter`，可能是 filters 或者 filters_script 的值，如果这两个字段都没有值，那就是空滤镜， `anull` 跟 `null`。

```
ost->avfilter = get_ost_filters(o, oc, ost);
```

![1-1](struct_outputstream\1-1.png)

**37，**`char *filters`，命令行 `-filter` 指定的滤镜字符串。

**38，**`char *filters_script`，滤镜脚本文件的名称以及路径。这是把滤镜的字符串放在一个文件里面，从文件里面读取。

---

```
AVDictionary *encoder_opts;
AVDictionary *sws_dict;
AVDictionary *swr_opts;
AVDictionary *resample_opts;
```

上面这些都是命令行设置的编码器参数，重采样参数，等等。

---

**39，**`char *apad`，命令行选项 `-apad` 的值，不清楚是做什么的，后面补充。

**40，**`OSTFinished finished`，默认是 0，有两个值，如下：

```
typedef enum {
    ENCODER_FINISHED = 1,
    MUXER_FINISHED = 2,
} OSTFinished ;
```

我也不太清楚，编码完成跟 muxer 完成有什么区别。

---

**41，**`int unavailable`，代表输出流不可用，可能只是暂时不可用，例如 av_read_frame() 从输入流返回 EAGAIN 错误码，就会导致输出流暂时不可用，暂时不可用就会休眠一段时间。

**42，**`int stream_copy`，如果命令行使用了 copy ，这个字段会置为 1。

**43，**`int initialized`，当前流是否已经被初始化了，只有初始化才能往容器里面写头部信息。

```
// init_output_stream() has been called for this stream
    // The encoder and the bitstream filters have been initialized and the stream
    // parameters are set in the AVStream.
    int initialized;
```

**44，**`int inputs_done`，代表没有找到输出流对应的输入流，通常不会没找到，所以通常一直都是 

**45，**`const char *attachment_filename`，命令行 `-attach` 功能的字段，添加附件用的。

**46，**`int copy_initial_nonkeyframes`，命令行 `-copyinkf` 功能的字段。

**47，**`int copy_prior_start`，命令行 `-copypriorss` 功能的字段。

**48，**`char *disposition`，命令行 `-disposition` 功能的字段。

**49，**`int keep_pix_fmt`，命令行如果指定了 pix_fmt，`keep_pix_fmt` 这个字段的值会是 1。

---

下面是一些统计相关的字段。

**50，**`uint64_t data_size`，由该流所有的 AVPacekt 的 data_size 字段累计起来的。

**51，**`uint64_t packets_written`，已经有多少个 AVPacket 被送进去了 muxer。

**52，**`uint64_t frames_encoded`，`uint64_t samples_encoded`，已经有多少个 AVFrame 被送进去了 编码器。

**53，**`int quality`，视频的编码器的编码质量，通过以下方式计算。

```
uint8_t *sd = av_packet_get_side_data(pkt, AV_PKT_DATA_QUALITY_STATS, NULL);
ost->quality = sd ? AV_RL32(sd) : -1;
```

**54，**`int max_muxing_queue_size`，muxing_queue 队列的大小，如果视频输出流已经初始化完成了，当音频输出流没有初始化完成的时候，这时候是不能 用 `avformat_write_header()` 写入 头信息到容器。

但是视频编码器确实输出了 AVPacket，这些 AVPacket 不能写入容器，就只能先放到 muxing_queue  队列里面缓存，等到音频流也解码出数据，音频输出流也初始化完成之后，这些 muxing_queue  队列缓存的 AVPacekt 才会写入容器。

这是 ffmepg.exe 转换器的逻辑，只有当所有输出流都初始化完成才写入头部。

**55，**`AVFifoBuffer *muxing_queue`，muxing_queue  队列。

**56，**`size_t muxing_queue_data_size`，muxing_queue  队列里面有多少数据了。

**57，**`size_t muxing_queue_data_threshold`，阈值，超过这个阈值会扩容 muxing_queue  队列的内存。

**58，**`int pict_type`，最近一次编码器输出的 AVPacket 的图片类型，例如 I 帧， P帧，还是 B 帧。

```
uint8_t *sd = av_packet_get_side_data(pkt, AV_PKT_DATA_QUALITY_STATS,NULL);
ost->pict_type = sd ? sd[4] : AV_PICTURE_TYPE_NONE;
```

**59，**`int64_t error[4]`，统计编码错误用的，后面补充。PSNR

---

你不需要一下子就弄懂 OutputStream 结构的各个字段的作用，我现在也没弄懂全部的字段。

大部分字段的作用不会相互干扰，所以你需要用到对应的命令行选项的时候再去看学习效率会更快一些。

本文那些没有讲得特别清楚的字段，后面另外会补充一个文章的链接，通过实际的命令行示例来分析其的作用。

---

参考文章：

1，《FFmpeg从入门到精通》刘歧， 赵文杰
