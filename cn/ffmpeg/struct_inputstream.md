# InputStream数据结构分析—ffmpeg.c源码分析

<div id="meta-description---">struct InputStream 是单个输入流的管理器。是由 add_input_stream() 函数申请内存，以及赋值 InputStream 的各个字段的。而 input_streams 数组是一个全局变量，包含了所有输入文件里面的所有输入流。</div>

`struct InputStream` 是**单个输入流的管理器**。是由 `add_input_stream()` 函数申请内存，以及赋值 `InputStream` 的各个字段的。

而 `input_streams` 数组是一个全局变量，包含了所有输入文件里面的所有输入流。

```
nputStream **input_streams = NULL;
int        nb_input_streams = 0;
```

你在二次开发 `ffmpeg.exe` 的时候，可以用 `input_streams` 全局变量来获取到所有的**输入流**。

`struct InputStream`数据结构 的定义如下：

```
typedef struct InputStream {
    int file_index;
    AVStream *st;
    int discard;             /* true if stream data should be discarded */
    int user_set_discard;
    int decoding_needed;     /* non zero if the packets must be decoded in 'raw_fifo', see DECODING_FOR_* */
#define DECODING_FOR_OST    1
#define DECODING_FOR_FILTER 2

    AVCodecContext *dec_ctx;
    const AVCodec *dec;
    AVFrame *decoded_frame;
    AVFrame *filter_frame; /* a ref of decoded_frame, to be sent to filters */
    AVPacket *pkt;

    int64_t       start;     /* time when read started */
    /* predicted dts of the next packet read for this stream or (when there are
     * several frames in a packet) of the next frame in current packet (in AV_TIME_BASE units) */
    int64_t       next_dts;
    int64_t       dts;       ///< dts of the last packet read for this stream (in AV_TIME_BASE units)

    int64_t       next_pts;  ///< synthetic pts for the next decode frame (in AV_TIME_BASE units)
    int64_t       pts;       ///< current pts of the decoded frame  (in AV_TIME_BASE units)
    int           wrap_correction_done;

    int64_t filter_in_rescale_delta_last;

    int64_t min_pts; /* pts with the smallest value in a current stream */
    int64_t max_pts; /* pts with the higher value in a current stream */

    // when forcing constant input framerate through -r,
    // this contains the pts that will be given to the next decoded frame
    int64_t cfr_next_pts;

    int64_t nb_samples; /* number of samples in the last decoded audio frame before looping */

    double ts_scale;
    int saw_first_ts;
    AVDictionary *decoder_opts;
    AVRational framerate;               /* framerate forced with -r */
    int top_field_first;
    int guess_layout_max;

    int autorotate;

    ...省略字幕相关字段...

    int dr1;

    /* decoded data from this stream goes into all those filters
     * currently video and audio only */
    InputFilter **filters;
    int        nb_filters;

    int reinit_filters;

    /* hwaccel options */
    ...省略硬件解码相关字段...

    /* hwaccel context */
   ...省略硬件解码相关字段...

    /* stats */
    // combined size of all the packets read
    uint64_t data_size;
    /* number of packets successfully read for this stream */
    uint64_t nb_packets;
    // number of frames/samples retrieved from the decoder
    uint64_t frames_decoded;
    uint64_t samples_decoded;

    int64_t *dts_buffer;
    int nb_dts_buffer;

    int got_output;
} InputStream;
```

各个字段的解析如下（省略了字幕相关的字段）：

**1，**`int file_index`，`input_files` 数组里面的下标，代表 `InputStream` 对应的 `InputFile`。

**2，**`AVStream *st`，`AVFormatContext` 里面的 `AVStream` 。

**3，**`int discard`，如果是 1 会丢弃读取到的 `AVPacket`，刚开始的时候都是 1。例如文件里面有多个视频流，会全部都把 `discard` 设置为 1，然后再从中选出质量最好的视频流，把它的 `discard` 设置为 0 。

或者当输出流需要这个输入流的时候，也会置为 0 ，如下：

```
if (source_index >= 0) {
    ost->sync_ist = input_streams[source_index];
    input_streams[source_index]->discard = 0;
    input_streams[source_index]->st->discard = input_streams[source_index]->user_set_discard;
}
```

因此，如果你用 `ffmpeg.exe` 对一个有多个视频流的文件进行转换，默认只会输出一个视频流。

**4，**`int user_set_discard`，命令行选项 `-discard` 的值，默认是 AVDISCARD_NONE，可选的值在 `AVDiscard` 枚举里面。

```
enum AVDiscard{
    /* We leave some space between them for extensions (drop some
     * keyframes for intra-only or drop just some bidir frames). */
    AVDISCARD_NONE    =-16, ///< discard nothing
    AVDISCARD_DEFAULT =  0, ///< discard useless packets like 0 size packets in avi
    AVDISCARD_NONREF  =  8, ///< discard all non reference
    AVDISCARD_BIDIR   = 16, ///< discard all bidirectional frames
    AVDISCARD_NONINTRA= 24, ///< discard all non intra frames
    AVDISCARD_NONKEY  = 32, ///< discard all frames except keyframes
    AVDISCARD_ALL     = 48, ///< discard all
};
```

**5，**`int decoding_needed`，大于 0 代表输入流需要进行解码操作，有两个值，DECODING_FOR_OST ，DECODING_FOR_FILTER

**6，**`AVCodecContext *dec_ctx`，解码器上下文/解码器实例，输入流的 `AVPacket` 会丢给解码实例进行解码。

**7，**`const AVCodec *dec`，解码器信息

**8，**`AVFrame *decoded_frame`，从解码器上下文 解码出来的 `AVFrame`

**9，**`AVFrame *filter_frame`，这个字段主要是用来增加引用计数的，可能在调用 `av_buffersrc_add_frame_flags()` 之后，AVFrame 的引用技术会减一，所以需要先复制一份引用，如下：

![1-1](struct_inputstream\1-1.png)

**10，**`AVPacket *pkt`，从 `av_read_frame()` 里面读出来的 `AVPacket`，只是复制了一下引用。

![1-2](struct_inputstream\1-2.png)

**11，**`int64_t start`，记录此流是什么时候开始处理的，主要是给 `-re` 选项使用的，模拟帧率速度，录制视频模拟直播用的。

**12，**`int64_t next_dts`，下一帧的 解码时间，这是通过计算得到的预估值，如果读取出来的 `AVPacket` 没有 `dts` ，会用这个值代替。

**13，**`int64_t dts`，最近一次从 `av_read_frame()` 读出来的 `AVPacket` 的 `dts` 

**14，**`int64_t next_pts`，下一个 `AVFrame` 的 pts，通过当前 `AVFrame` 的 pts 加上 duration 计算出来的，应该是给一些没有 pts 值的 `AVFrame` 用的。

**15，**`int64_t pts`，最近一次从解码器里面解码出来的 `AVFrame` 的 pts，记录这个值主要是给 `send_filter_eof()` 用的，防止回滚。

![1-3](struct_inputstream\1-3.png)

**15，**`int64_t wrap_correction_done`，在一些流格式，例如 TS，会有时间戳环回的问题，`int64` 是有大小限制的，超过这个大小会环回，`ffmpeg.exe` 需要处理环回的逻辑。

**16，**`int64_t filter_in_rescale_delta_last`，专门给音频帧转换时间基用的，因为音频的连续性太强，如果有些许误差需要保存下来，在下次转换的时候进行补偿。

![1-4](struct_inputstream\1-4.png)

**17，**`int64_t min_pts`， 从 `av_read_frame()` 读出来的 `AVPacket` 的最小 `pts` ，**不一定是第一帧的 pts**，因为命令行参数通过 `-ss` 选项，指定从哪里开始处理。

**18，**`int64_t max_pts`，从 `av_read_frame()` 读出来的 `AVPacket` 的最大 `pts` ，**不一定是最后一帧的 pts**，因为命令行参数通过 `-t` 选项，指定只处理多久的时长。

**19，**`int64_t cfr_next_pts`，给 `-r` 选项用的。

**20，**`int64_t nb_samples`，记录的是最近一次解码出来的 `AVFrame` 里面的 `nb_samples`。用途我也不清楚，后面补充。

**21，**`double ts_scale`，默认值是 1.0，可通过命令行选项 `-itsscale` 进行改变，主要作用是将 pts ，dts 进行放大，放大的方法是乘以 `ts_scale`。

**22，**`int saw_first_ts`，标记是不是已经读取到属于该流的第一个 `AVPacket`

**23，**`AVDictionary *decoder_opts`，从 `OptionsContext` 里面转移过来的解码器参数。

![1-5](struct_inputstream\1-5.png)

**24，**`AVRational framerate`，命令行选项 `-r` 的值。

**25，**`int top_field_first`，命令行选项 `-top` 的值，好像是给隔行扫描视频用的。

**26，**`int guess_layout_max`，命令行选项 `-guess_layout_max` 的值，猜测的最大的声道布局。

**27，**`int autorotate`，命令行选项 `-autorotate` 的值，是否插入纠正旋转的 filter 滤镜，有些视频的画面中途会上下旋转，设置这个选项可以自动纠正旋转，不会上下颠倒。

**28，**`int dr1`，用 Find Useage 找不到使用这个字段的代码，所以是没用的字段。

**29，**`InputFilter **filters` 跟 `int nb_filters`，输入流绑定的 入口滤镜，可以是绑定多个入口滤镜的，输入流解码出来的 `AVFrame` 会往多个入口滤镜发送。

**30，**`int reinit_filters`，命令行选项 `-reinit_filter` 的值，0 代表输入信息如果产生变化也不重新初始化 `FilterGraph`，输入信息产生变化是指 解码出来的 `AVFrame` 的宽高中途变大或者变小之类的。`reinit_filters` 默认是 -1，就是会重新初始化 `FilterGraph`。

---

下面这些都是统计相关的字段。

**31，**`uint64_t data_size`，从 `av_read_frame()` 读出来的 `AVPacket` 的 size 的总和。也就是统计当前流一共读出出来了多少字节的数据。

**32，**`uint64_t nb_packets`，从 `av_read_frame()` 读出来的 `AVPacket` 数量的总和。

**33，**`uint64_t frames_decoded`，`uint64_t samples_decoded`，从解码器解码出来的 `AVFrame` 的数量。

**34，**`int64_t *dts_buffer`，`int nb_dts_buffer`，我没看懂这两个字段干什么的，可能是旧代码没有删除。

**35，**`int got_output`，只要解码器已经解码出一帧 `AVFrame`，这个字段就会置为 1。



