# audio_thread音频解码线程分析—ffplay.c源码分析

<div id="meta-description---">audio_thread线程实际上就是音频解码线程，audio_thread线程主要是负责解码PacketQueue队列里面的AVPacket的，解码出来 AVFrame，然后丢给入口滤镜，再从出口滤镜把 AVFrame读出来，再插入FrameQueue队列。</div>

之前在 `stream_component_open()` 里面的 `decode_start()` 函数开启了 `audio_thread` 线程，如下：

![1-1](audio_thread\1-1.png)

`audio_thread` 线程主要是负责 **解码** `PacketQueue` 队列里面的 `AVPacket` 的，解码出来 `AVFrame`，然后丢给**入口滤镜**，再从**出口滤镜**把 `AVFrame` 读出来，再插入 `FrameQueue` 队列。

流程图如下：

![1-2](audio_thread\1-2.jpg)



---

如上，`audio_thread` 函数一开始就进入 一个 `do{...}while{...}` ，不断地调 `decoder_decode_frame()` 函数来解码出 `AVFrame`，然后把 `AVFrame` 往 入口滤镜 丢，再**循环**调 `av_buffersink_get_frame_flags()`，不断从出口滤镜收割经过 `Filter` 的`AVFrame`，最后调 `frame_queue_push()` 把 `AVFrame` 插入 `FrameQueue` 队列。

但是如果解码出来的 `AVFrame` 的音频格式与**入口滤镜**要求的音频格式不一样，会**重建滤镜**（`reconfigure`），如下：

![1-2-2](audio_thread\1-2-2.png)

首先，开始的**入口滤镜 （audio_filter_src）的音频格式 **是直接从**解码器实例（avctx）**里面取的，如下：

```
ffplay.c 2648行
is->audio_filter_src.freq           = avctx->sample_rate;
is->audio_filter_src.channels       = avctx->channels;
is->audio_filter_src.channel_layout = get_valid_channel_layout(avctx->channel_layout, avctx->channels);
is->audio_filter_src.fmt            = avctx->sample_fmt;
```

而**解码器实例（avctx）**的音频信息又是从 **容器层**的流信息里面取出来的。

```
ffpaly.c 2592 行
ret = avcodec_parameters_to_context(avctx, ic->streams[stream_index]->codecpar);
```

所以以下 3 种场景会重新创建滤镜：

**1，**容器层记录的采样率等信息是错误的，与实际解码出来的不符。

**2，**解码过程中，中途解码出来的 `AVFrame` 的采样率，声道数或者采样格式 出现变动，与上一次解码出来的 `AVFrame` 不一样。

**3，**进行了快进快退操作，因为快进快退会导致 `is->auddec.pkt_serial` 递增。详情请阅读《[FFplay序列号分析](https://ffmpeg.xianwaizhiyin.net/ffplay/serial.html)》。我也不知道为什么序列号变了要重建滤镜。

这3种情况，`ffplay` 都会处理，只要解码出来的 `AVFrame` 跟入口滤镜的格式不一致，都会重建滤镜，把入口滤镜的格式设置为当前的 `AVFrame` 的格式，这样滤镜处理才不会出错。

补充：`last_serial` 变量一开始是 `-1`，而  `is->auddec.pkt_serial` 一开始是 0，所以一开始是必然会执行一次 `reconfigure` 操作。

由于每次读取**出口滤镜**的数据，都会用 `while` 循环把缓存刷完，不会留数据在滤镜容器里面，所以重建滤镜不会导致音频数据丢失。我圈一下代码里面的重点，如下：

![2-1](audio_thread\2-1.png)

------

`audio_thread` 线程的逻辑比较简单，复杂的地方都封装在它调用的**子函数**里面，所以本文简单讲解一下，`audio_thread()` 里面调用的各个函数的作用。

**1，**`decoder_decode_frame()`，从 `PacketQueue` 里面解码出来 `AVFrame`，此函数会**阻塞**，直到解码出来 `AVFrame`，或者返回错误。这个函数有 3 个返回值。

- 返回 1，获取到 `AVFrame` 。
- 返回 0 ，获取不到 `AVFrame` ，0 代表已经解码完MP4的所有`AVPacket`。这种情况一般是 `ffplay` 播放完了整个 MP4 文件，窗口画面停在最后一帧。但是由于你可以按 C 键重新循环播放，所以即便返回 0 也不能退出 `audio_thread` 线程。
- 返回 -1，代表 `PacketQueue` 队列关闭了（`abort_request`）。返回 `-1`  会导致 `audio_thread()` 函数用 `goto the_end` 跳出 `do{}whlle{}` 循环，跳出循环之后，`audio_thread` 线程就会自己结束了。返回 `-1` 通常是因为关闭了 `ffplay` 播放器。

更详细的分析请阅读《[decoder_decode_frame函数分析](https://ffmpeg.xianwaizhiyin.net/ffplay/decoder_decode_frame.html)》。

**2，**`configure_audio_filters()`，创建音频滤镜函数，之前在《[FFplay音频滤镜分析](https://ffmpeg.xianwaizhiyin.net/ffplay/configure_audio_filters.html)》已经讲过此函数。

**3，**`av_buffersrc_add_frame()`，往入口滤镜发送 `AVFrame`。

**4，**`av_buffersink_get_frame_flags()`，从出口滤镜读取 `AVFrame`。

滤镜相关的函数推荐阅读 **FFmpeg实战之路** 一章的 《[FFmpeg的scale滤镜介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/scale.html)》

**5，**`frame_queue_peek_writable()`，从 `FrameQueue` 里面取一个可以写的 `Frame` 出来。此函数也可能会**阻塞**。

**6，**`frame_queue_push()`，这个函数有点奇怪，他其实不是把之前的 Frame 塞进去队列，而是把队列的写索引值 +1。

跟 `FrameQueue`队列相关的函数都在 《[FrameQueue队列分析](https://ffmpeg.xianwaizhiyin.net/ffplay/frame_queue.html)》一文中。

------

`audio_thread()` 函数最后还有一个重点，就是当 出口滤镜 **结束**的时候，`finished` 就会设置为 非 0 。

```
if (ret == AVERROR_EOF)
	is->auddec.finished = is->auddec.pkt_serial;
```

提示：只有往入口滤镜发送了 `NULL` 的 `AVFrame` ，出口滤镜才会结束。**读完数据** 跟 **结束**是两种状态，读完代表滤镜暂时没有数据可读，但是只要再往入口滤镜 发 `AVFrame`，出口滤镜就会又有数据可读。

而结束，代表不会再有 `AVFrame` 往入口滤镜发。

------

至此，`audio_thread()` 线程分析完毕。
