# FFmpeg写入输出文件—FFmpeg API教程

<div id="meta-description---">本文介绍如何使用 FFmpeg 的 API 函数 avio_open2 打开一个输出文件，然后用 av_interleaved_write_frame 来把编码器输出的 AVPacket 保存进去文件。</div>

本文介绍如何使用 FFmpeg 的 API 函数 `avio_open2` 打开一个输出文件，然后用 `av_interleaved_write_frame` 来把编码器输出的 `AVPacket` 保存进去文件。

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/output )，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

------

与 输出 相关的 API 函数如下：

1，`avformat_alloc_output_context2`，申请一个输出文件上下文，这个函数会生成 `AVFormatContext`

2，`avformat_new_stream`，往容器增加一个**输出流**，可以是音频流，或者视频流，或者一些其他的数据流。

3，`avcodec_parameters_from_context`，把 `AVCodecContext` 里面的 **编解码参数** 复制到**输出流**的 `AVCodecParameters`  。

4，`avio_open2`，正式打开输出文件。

5，`avformat_write_header`，往输出文件写入头信息。

6，`av_interleaved_write_frame`，把 `AVPacket` 写入输出文件。

------

要特别注意 `avformat_write_header` 这个函数，必须是容器里面的所有流都初始化完成了，才能调 `avformat_write_header`。

本文的代码只有一个视频流，所以比较简单，但是如果同时有音频流跟视频流，那就必须等音频也解码出 `AVFrame`，音频流初始化完成才能 执行 `avformat_write_header`。

输出流的初始化主要通过两个函数完成，`avformat_new_stream` 跟 `avcodec_parameters_from_context`。

`avformat_new_stream` 函数是创建一个输出流，而 `avcodec_parameters_from_context` 是把编码器参数复制到流里面，因为容器里面也需要保存一下编码器的参数跟信息。

我经常会把 容器，输出文件上下文 混着来说，其实是一个意思，MP4 是一个 容器，可以容纳音频流视频流。

------

下面就结合本文的代码来理解一下上面这些函数跟结构体的使用。

![output-1-1](output\output-1-1.png)

上面的代码，是申请一个输出文件的上下文，然后添加了一个流，这里还没确定这个流是音频流还是视频流，是通过后面的 `avcodec_parameters_from_context` 来确定的，会设置 `codec_type` 。

还需要特别注意这行代码，：

```
st->time_base = fmt_ctx->streams[0]->time_base;
```

我把输入流的时间基 复制 给了输出流。

------

继续看代码，如下：

![output-1-2](output\output-1-2.png)

上面代码的调用流程如下：

`avcodec_parameters_from_context` ➔ `avio_open2`  ➔ `avformat_write_header`

------

继续看代码，如下：

![output-1-3](output\output-1-3.png)

编码器输出 `AVPakcet` 之后，在写入文件之前，还需要做2个操作。

1，设置 `AVPakcet` 的 `stream_index`，因为编码器输出的 `AVPakcet`  还没有跟输出流绑定，容器里面可以有多个流，所以写入的时候需要根据 `stream_index` 来确认这个包应该写进去哪个流里面。

2，转换 `AVPakcet` 的 PTS ，DTS，duration 的时间基，因为本文输入流跟输出流的时间基一样，所以不转换也可以。

但是如果输入流的时间基跟输出流的时间基不一样就需要转换，因为当时设置编码器的时间基是以输入流为准的，所以编码器输出的 `AVPacket` 的时间基也是输入流的时间基。所以需要把  `AVPacket` 的时间基转成 输出流的时间基。

------

最后调 `av_interleaved_write_frame` 写入文件即可。
