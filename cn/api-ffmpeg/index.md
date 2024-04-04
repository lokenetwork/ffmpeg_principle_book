# FFmpeg基础API教程

<div id="meta-description---">本章通过一些小的代码示例 FFmpeg 里面一些基本常用的数据结构 AVFormatContext,AVPacket,AVCodec,AVCodecContext 跟 API 函数的使用，一些比较生僻的数据结构跟函数会在后续的 ffmpeg.c 源码分析一章继续讲解。</div>

本章通过一些小的代码示例介绍 FFmpeg 里面一些**基本常用**的数据结构 跟 API 函数的使用。

学习 FFmpeg 的使用是比较有难度的，因为音视频封装格式，编码格式纷繁复杂，ffmpeg 里面有较多代码是处理异常情况的。**本章节的所有代码，为了便于初学者学习，删掉了一些时间戳处理代码、错误处理代码 等等**，所以不建议在正式环境使用这些代码，相同的功能，建议你直接从 `ffmpeg.c` ，`ffplay.c` ，`ffprobe.c` 这 3个文件里面抄代码，他们的拥有更好的兼容性跟鲁棒性。

本章使用的编译环境是 qt creator 跟 msvc。请参考以下文章搭建好开发环境。

- [Qt安装教程](https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/qt-install.html)
- [Qt使用FFmpeg的动态库](https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/qt-shared.html)
- [Qt使用FFmpeg的静态库](https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/qt-static.html)

如果你需要把这些 Qt 示例移植到 clion 的环境调试，推荐阅读《[移植Qt示例到clion调试](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/qt_to_clion.html)》

------

**常用的结构体如下**：

1，`AVFormatContext` ，容器格式上下文，或者叫封装格式上下文。可以理解为 MP4 或者 FLV 的容器。

2，`AVPacket`，数据包（已编码压缩的数据），这里面的数据通常是一帧视频的数据，或者一帧音频的数据。

`AVPacket` 他本身是没有编码数据的，他只是管理编码数据。

3，`AVCodecContext`，这个结构体可以是 **编码器** 的上下文，也可以是 **解码器** 的上下文，两者使用的是同一种数据结构。

4，`AVCodec`，编解码信息。

5，`AVCodecParameters`，编解码参数。

6，`AVFrame` ，解码之后的 YUV 数据。`AVFrame` 跟 `AVPacket` 类似，都是一个管理数据的结构体，他们本身是没有数据的，只是引用了数据。

7，`AVDictionary`，一个传递参数的通用数据结构。这个结构体主要用来传递参数。

------

**常用的 API 函数如下**：

1，`avformat_open_input`，打开输入文件，这个函数会生成 `AVFormatContext` 。打开输入文件有两种方式，`char *filename` 或者 `AVInputFormat *fmt`

2，`avformat_find_stream_info` ，探测函数，读取输入文件的一部分信息来分析出 各个流的情况。

3，`av_packet_alloc`，初始化一个 `AVPacket`。

4，`av_read_frame`，从 `AVFormatContext` 容器里面读取一个 `AVPacket`，需要注意，虽然函数名是 `frame`，但是读取的是 `AVPacket`。

5，`av_packet_unref`，减少 `AVPacket` 对 编码数据的引用次数。减到 0 会释放 编码数据的内存。

6，`av_packet_free`，释放 `AVPacket` 自身的内存。里面会调 `av_packet_unref`

------

1，`avcodec_alloc_context3`，通过传递 `AVCodec` **编解码信息**来初始化上下文。

2，`avcodec_parameters_to_context`，把流的 `AVCodecParameters` 里面的 **编解码参数** 复制到 `AVCodecContext` 。

3，`avcodec_open2`，打开一个编码器 或者 解码器。

4，`avcodec_send_packet`，往 `AVCodecContext` 解码器 发送一个 `AVPacket` 。

5，`avcodec_receive_frame`，从 `AVCodecContext` 解码器 读取一个 `AVFrame`。

------

1，`avformat_alloc_output_context2`，申请一个输出文件上下文，这个函数会生成 `AVFormatContext`

2，`avformat_new_stream`，往容器增加一个**输出流**，可以是音频流，或者视频流，或者一些其他的数据流。

3，`avcodec_parameters_from_context`，把 `AVCodecContext` 里面的 **编解码参数** 复制到**输出流**的 `AVCodecParameters` 。

4，`avio_open2`，正式打开输出文件。

5，`avformat_write_header`，往输出文件写入头信息。

6，`av_interleaved_write_frame`，把 `AVPacket` 写入输出文件。

------

这里我说一下自己个人一些偏面的想法，`FFmpeg` 的 API 函数的设计，有些地方不是特别好的，他没有对称性，例如有 `avformat_open_input` 是打开输入文件，却没有一个 `avformat_open_ouput` 函数来打开输出文件。

可能是我对 `FFmpeg` 的整体还不太理解。可能存在一些无法逾越的障碍来实现对称性。

------

参考资料：

1，[《ffmpeg tutorial》](http://dranger.com/ffmpeg/ffmpegtutorial_all.html#tutorial06.html) -  dranger





