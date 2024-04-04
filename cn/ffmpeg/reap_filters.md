# reap_filters收割滤镜—ffmpeg.c源码分析

<div id="meta-description---">reap_filters() 会收割（reap）所有输出滤镜（buffer sink），从 buffer sink 读取到 AVFrame 之后，就会发送给编码器进行编码，如果编码器有 AVPacket 出来，就会进行 muxer 操作，进行封装，写入文件保存。</div>

`reap_filters()` 虽然它的名称叫 filter，但是它也是一个 进行编码，`muxer` 操作的**总函数**。

**`reap_filters()` 会收割（reap）所有输出滤镜（buffer sink），从 buffer sink 读取到 `AVFrame` 之后，就会发送给编码器进行编码，如果编码器有 `AVPacket` 出来，就会进行 `muxer` 操作，进行封装，写入文件保存。**

`reap_filters()` 函数的定义如下：

```
/**
 * Get and encode new output from any of the filtergraphs, without causing
 * activity.
 *
 * @return  0 for success, <0 for severe errors
 */
static int reap_filters(int flush)
```

这个函数只有一个参数 `int flush`，`flush` 如果是 1，会导致 `do_video_out()` 最后一个参数传的是 NULL，如下：

![1-1](reap_filters\1-1.png)

 我也不太清楚 `do_video_out()` 函数的第三个参数是 NULL 有什么作用。

---

`reap_filters()` 函数的流程图如下：

![1-2](reap_filters\1-2.jpg)

当滤镜容器还未打开的时候，是不会执行收割操作的，如下，直接退出 `reap_filters()` 函数。

![1-3](reap_filters\1-3.jpg)

---

`reap_filters()` 函数的流程图也有一个重点，就是初始化音频的输出流，与初始化视频的输出流的时机是不一样的。

音频的输出流可以在未从滤镜读取到 `AVFrame` 的时候就开始初始化，而视频的输出流的初始化，需要从滤镜拿到 `AVFrame` 之后才能初始化，视频的初始化是在 封装在`do_video_out()` 函数里面的。

至于为什么音频输出流需要提前初始化，可以看一下他的注释，我没细看。

```
/*
* Unlike video, with audio the audio frame size matters.
* Currently we are fully reliant on the lavfi filter chain to
* do the buffering deed for us, and thus the frame size parameter
* needs to be set accordingly. Where does one get the required
* frame size? From the initialized AVCodecContext of an audio
* encoder. Thus, if we have gotten to an audio stream, initialize
* the encoder earlier than receiving the first AVFrame.
*/
if (av_buffersink_get_type(filter) == AVMEDIA_TYPE_AUDIO)
	init_output_stream_wrapper(ost, NULL, 1);
```

![1-4](reap_filters\1-4.png)

---

`av_buffersink_get_frame_flags()` 从 `buffersink` 滤镜里面读取到 `AVFrame` 之后，就会丢给 `do_video_out()` 或者 `do_audio_out()` 进行处理。

`do_video_out()` 主要的职责是对**视频** `AVFrame` 进行编码，然后 `muxer` 封装写入文件保存，但是 `do_video_out()` 有帧率变换的逻辑，也就是命令行 -r 选项的功能代码，所以 `do_video_out()` 会非常复杂，推荐阅读《[do_video_out编码封装](https://ffmpeg.xianwaizhiyin.net/ffmpeg/do_video_out.html)》

`do_audio_out()` 主要的职责是对**音频** `AVFrame` 进行编码，然后 `muxer` 封装写入文件保存，推荐阅读《[do_audio_out编码封装](https://ffmpeg.xianwaizhiyin.net/ffmpeg/do_audio_out.html)》

---

`reap_filters()` 函数会在 `while(1){...}` 里面不断循环调用 `av_buffersink_get_frame_flags()` 读取数据，直到没有数据能读出来才跳出循环。

这个过程就是收割（reap）。一点都不剩的收割。





