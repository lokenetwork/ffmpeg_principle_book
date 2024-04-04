# FFmpeg丢弃音频流—FFmpeg API教程

<div id="meta-description---">FFmpeg丢弃音频流</div>

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/discard_audio)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

------

我们在处理音视频文件的时候，有时候仅仅需要处理视频，不需要处理音频，但是正常调用 `av_read_frame` 还是会返回音频数据包。如果我们要丢弃音频包，可以有两种做法：

1，`av_read_frame` 读取到 `AVPacket` ，判断 `AVPacket` 的数据类型，如果是音频，就丢弃。如果是视频，就解码。

2，设置 `AVStream` 的 `discard` 属性为 `AVDISCARD_ALL`，这样 `av_read_frame` 就不会返回 这个流的 `AVPacket`。

第一种方法比较简单，所以不做过多讲解，第二种方法可能会带来少许的性能提升，第二种方法的示例代码如下：

![1-1](discard_stream\1-1.png)

可以看到，如果设置了 `discard` 属性为 `AVDISCARD_ALL`，`av_read_frame` 函数就不会返回 `stream_index` 为 1 的 `AVPacket`。

在 本文的 `juren-30s.mp4` 示例文件，`stream[1]` 就是音频流，而 `stream[0]` 是视频流。

------

即便设置了`AVDISCARD_ALL`，在一种情况里面还是会返回 `stream[1]`  的 `AVPacket`，这种情况就是，你之前曾经调过 `find_stream_info`，`find_stream_info` 这个函数会读取数据包来分析流信息，但是他读取之后不是丢弃，而是会放进去缓存里面，`av_read_frame` 的时候就可以直接从内存直接拿到缓存数据，不需要读磁盘文件。

就因为在缓存里面有数据，即便你设置了 `AVDISCARD_ALL`，还是会返回一小部分 `stream[0]`  的 `AVPacket`，只是一小部分。读完缓存就不会再读。

我也不清楚这个是 FFmpeg 的 bug 还是 **特性**。









