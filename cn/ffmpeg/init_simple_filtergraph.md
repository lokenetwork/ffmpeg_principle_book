# init_simple_filtergraph初始化简单滤镜—ffmpeg.c源码分析

<div id="meta-description---">init_simple_filtergraph初始化简单滤镜</div>

在 `new_video_stream()` 跟 `new_audio_stream()` 创建完视频流跟音频流之后，就会初始化简单滤镜，如下：

![0-1](init_simple_filtergraph\0-1.jpg)

只要命令行没有使用 `-filter_complex` 选项，都是跑的简单滤镜的逻辑。

---

`init_simple_filtergraph()` 函数的代码比较简单，只有 20多行，就是一些申请内存，绑定赋值操作。

重要的是它里面使用到的数据结构，`struct FilterGraph`，`struct InputFilter`，`struct OutputFilter`。

**`FilterGraph` 是容器，而 `InputFilter`，`OutputFilter` 是连接 输入流 与 输出流 的桥梁，他们是桥梁。**

---

**`struct FilterGraph` 的定义如下：**

```
typedef struct FilterGraph {
    int            index;
    const char    *graph_desc;

    AVFilterGraph *graph;
    int reconfiguration;

    InputFilter   **inputs;
    int          nb_inputs;
    OutputFilter **outputs;
    int         nb_outputs;
} FilterGraph;
```

```
FilterGraph **filtergraphs;
int        nb_filtergraphs;
```

`FilterGraph` 的各个字段的解释如下：

**1，**`int index`，全局变量 `filtergraphs` 数组里的索引，`ffmpeg.exe` 里面是可以有多个 `FilterGraph` 的，例如音频的 `FilterGraph` ，视频的 `FilterGraph` 。

视频也可以定义多个 `FilterGraph` ，视频的多个 `FilterGraph`  我没找到命令行例子，后续补充。

**2，**`const char *graph_desc`，命令行选项  `-filter_complex`  的值，如果是简单滤镜，这个字段是个空字符串。

**3，**`AVFilterGraph *graph`，FFmpeg 的数据结构，滤镜容器。

**4，**`int reconfiguration`，我也不太了解这个字段的作用，只搜索到 `configure_input_audio_filter()` 函数里面有使用这个字段。

**5，**`InputFilter **inputs`，`buffer` 入口滤镜数组，入口滤镜可以有多个

**6，**`OutputFilter **outputs`，`buffersink` 出口滤镜数组,，出口滤镜可以有多个

---

**`struct InputFilter` 的定义如下：**

```
typedef struct InputFilter {
    AVFilterContext    *filter;
    struct InputStream *ist;
    struct FilterGraph *graph;
    uint8_t            *name;
    enum AVMediaType    type;   // AVMEDIA_TYPE_SUBTITLE for sub2video

    AVFifoBuffer *frame_queue;

    // parameters configured for this input
    int format;

    int width, height;
    AVRational sample_aspect_ratio;

    int sample_rate;
    int channels;
    uint64_t channel_layout;

    AVBufferRef *hw_frames_ctx;

    int eof;
} InputFilter;
```

`InputFilter` 的各个字段的解释如下：

**1，**`AVFilterContext *filter`，滤镜上下文，我把它称为滤镜实例，这个字段肯定是一个 buffer入口滤镜。是通过 `avfilter_graph_create_filter()` 创建的。

**2，**`struct InputStream *ist`，关联的输入流，这个入口滤镜的 `AVFrame` 是从哪个输入流输入的。这里注意，多个 `InputFilter` 可能是由同一个输入流来输入的。

**3，**`struct FilterGraph *graph`，关联的 `FilterGraph` 。

**4，**`uint8_t *name`，名称，通过 `describe_filter_link()` 计算出来的。

**5，**`enum AVMediaType type`，数据类型，可以是视频，音频，或者字幕。但是复杂滤镜目前不支持字幕。

**6，**`AVFifoBuffer *frame_queue`，AVFrame 的缓存队列，当输入流解码出来 `AVFrame`，但是还不能调 `av_buffersrc_add_frame_flags()` 发往滤镜，就会先缓存在  `frame_queue` 里面。

ffmpeg.exe 转换器的逻辑是这样的，必须所有的输入 `InputFilter` 都初始化完成之后，才能把 `AVFrame` 写入 `FilterGraph`。

而 `InputFilter` 的初始化，需要等到解码出 `AVFrame`，是所有的流都解码出 `AVFrame`。

`InputFilter::format` 不等于 -1 就是 初始化完成了，`InputFilter::format` 是从 解码出来的 AVFrame 赋值过来的。

具体可以看到 `ifilter_parameters_from_frame()` 函数，如下：

```
int ifilter_parameters_from_frame(InputFilter *ifilter, const AVFrame *frame)
{
    av_buffer_unref(&ifilter->hw_frames_ctx);

    ifilter->format = frame->format;
    ...省略代码...
}
```

**7，**`int format`，格式，输入流 AVFrame 的格式。

后面的这些字段，是音频，视频相关的字段，还有硬件滤镜上下文件（hw_frames_ctx），**部分滤镜可以用硬件加速**。

```
int width, height;
AVRational sample_aspect_ratio;

int sample_rate;
int channels;
uint64_t channel_layout;

AVBufferRef *hw_frames_ctx;

int eof;
```

---

**`struct OutputFilter` 的定义如下：**

```
typedef struct OutputFilter {
    AVFilterContext     *filter;
    struct OutputStream *ost;
    struct FilterGraph  *graph;
    uint8_t             *name;

    /* temporary storage until stream maps are processed */
    AVFilterInOut       *out_tmp;
    enum AVMediaType     type;

    /* desired output stream properties */
    int width, height;
    AVRational frame_rate;
    int format;
    int sample_rate;
    uint64_t channel_layout;

    // those are only set if no format is specified and the encoder gives us multiple options
    int *formats;
    uint64_t *channel_layouts;
    int *sample_rates;
} OutputFilter;
```

`OutputFilter` 只讲一些**重点字段**，其他的都是比较容易懂的。

**1，**`AVFilterContext *filter`，滤镜上下文，这个字段肯定是一个 `buffersink` 出口滤镜。是通过 `avfilter_graph_create_filter()` 创建的

**2，**`struct OutputStream *ost`，关联的输出流

**3，**`struct FilterGraph *graph`，关联的 `FilterGraph` 

**4，**`uint8_t *name`，名称

---

由于 `FilterGraph->inputs` 跟 `FilterGraph->outputs` 是一个数组，**所以 入口滤镜 跟 出口滤镜 都是是可以有多个的**，例如下面的命令，定义了多个输入：

```
./ffmpeg -re -i input1.mp4 -re -i input2.mp4 -re -i input3.mp4 -re -i input4.mp4 
-filter_complex 
"nullsrc=size=640x480[base];
[0:v]setpts=PTS-STARTPTS,scale=320x240[upperleft];
[1:v]setpts=PTS-STARTPTS,scale=320x240[upperright];
[2:v]setpts=PTS-STARTPTS,scale=320x240[lowerleft];
[3:v]setpts=PTS-STARTPTS,scale=320x240[lowerright];
[base][upperleft]overlay=shortest=1[tmp1];
[tmp1][upperright]overlay=shortest=1:x=320[tmp2];
[tmp2][lowerleft]overlay=shortest=1:y=240[tmp3];
[tmp3][lowerright]overlay=shortest=1:x=320:y=240"
-c:v libx264 output.flv
```

`ffmpeg.exe` 它实现上面这条命令的功能，这条命令有 4 个输入，`[0:v]` ~ `[3:v]`，所以它在解析复杂滤镜的时候，会**创建 4 个 buffer 入口滤镜**来连接 `[0:v]` ~ `[3:v]`。

但是如果是简单滤镜，只会创建一个 buffer 入口滤镜。

但是 一个输入流可以连接多个 `InputFilter`，也可以只连接一个，所以其实上面的命令可以这样写。

```
./ffmpeg -re -i input1.mp4 -re -i input2.mp4 -re -i input3.m2t -re -i input4.mp4 
-filter_complex 
"nullsrc=size=640x480[base];
[0:v]setpts=PTS-STARTPTS,scale=320x240[upperleft];
[0:v]setpts=PTS-STARTPTS,scale=320x240[upperright];
[0:v]setpts=PTS-STARTPTS,scale=320x240[lowerleft];
[0:v]setpts=PTS-STARTPTS,scale=320x240[lowerright];
[base][upperleft]overlay=shortest=1[tmp1];
[tmp1][upperright]overlay=shortest=1:x=320[tmp2];
[tmp2][lowerleft]overlay=shortest=1:y=240[tmp3];
[tmp3][lowerright]overlay=shortest=1:x=320:y=240"
-c:v libx264 output.flv
```

只取**第一个文件的视频流**来输入，一个视频流输入给 4 个 buffer 入口滤镜，这样也是可以的。

所以，整个数据流的图如下：

![0-2](init_simple_filtergraph\0-2.jpg)

`OutputFilter` 也是一样的逻辑，只是多个输出的命令行我没找到，后面补充。

----

**上面说过，`InputFilter`，`OutputFilter` 是连接 输入流 与 输出流 的桥梁。**

所以 `init_simple_filtergraph()` 执行完毕之后，整个关系图如下：

![0-3](init_simple_filtergraph\0-3.jpg)



注意，虽然`init_simple_filtergraph()` 里面申请了 `InputFilter` 跟 `OutputFilter` 的内存，但是里面的滤镜上下文（AVFilterContext）还是 NULL，还没开始创建的。

从上面关系图可以看到，`InputFilter` 输入滤镜 与 `InputStream` 输入流 是双向绑定的，输出滤镜与输出流也是双向绑定的。

**但是一个输入流可以绑定多个滤镜，可以看到 `InputStream` 里面的 `filters` 字段是一个二级指针，一个数组。**

无论是 `FilterGraph`（滤镜容器），`InputFile`（输入文件），`OutputFile`（输出文件），`InputStream`（输入流），`OutputStream`（输出流），他们都有一个全局的数组来管理的，如下：

```
InputStream **input_streams = NULL;
int        nb_input_streams = 0;
InputFile   **input_files   = NULL;
int        nb_input_files   = 0;

OutputStream **output_streams = NULL;
int         nb_output_streams = 0;
OutputFile   **output_files   = NULL;
int         nb_output_files   = 0;

FilterGraph **filtergraphs;
int        nb_filtergraphs;
```

无论是 **复杂滤镜**，还是 **简单滤镜**，他们其实是共用同一种数据结构，同一种关系图，也可以说是同一种逻辑。

到这里，我必须趁热打铁，剧透，提前讲一些知识点。

其实上面那个关系图，就是 `ffmpeg.exe` 整体的转换流程，我把一些不太重要的箭头删掉，你就清楚了。

![0-4](init_simple_filtergraph\0-4.jpg)

先抛一个问题，如果命令行里面指定了多个输入流，而且输出流也有多个，例如有 一个视频流，一个音频流，怎么防止音频写入了很多数据，而视频流还没写入？

**答：**通过 输出流当前 的 `cur_dts` 来判断，这个字段代表当前输出流已经输出了多少时间的数据，`choose_output()` 函数会选出输出时间最小的一个流来进行处理。

没错，`ffmpeg.exe` 转换器，他不是遍历每个输入文件，来读取数据的。而是先用 `choose_output()` 确定输出流是那个，再根据输出流找到关联的输出滤镜，再根据输出滤镜找到 滤镜容器 `FilterGraph`。

然后再遍历 `FilterGraph` 里面的**输入滤镜**，用 `av_buffersrc_get_nb_failed_requests()` 函数找到失败次数最多的输入滤镜。

找到失败次数最多的输入滤镜之后，就可以找到输入滤镜关联的输入流，进而找到输入文件 `InputFile`，就可以调 `av_read_frame()` 来从文件读取 `AVPacket` 了。

但是，读取到的 `AVPacket` 不一定就是那个输入流的，因为一个文件里面可能有多个流，不过这已经是比较好的方法了。至少可以把范围缩小到一个文件里面。

整体的函数调用如下：

![0-5](init_simple_filtergraph\0-5.jpg)

总结一下，这个过程是通过 `cur_dts` 确定出哪个输出流的时间最小，反向找到最适合的输入流来读取数据。

无论是复杂滤镜，还是简单滤镜，这个逻辑都是一样的。



