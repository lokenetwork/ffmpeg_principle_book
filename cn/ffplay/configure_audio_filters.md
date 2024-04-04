# FFplay音频滤镜分析—ffplay.c源码分析

<div id="meta-description---"> configure_audio_filters() 函数的作用主要是配置 音频流的 滤镜，因为 ffplay 为了代码的通用性，即便命令行参数不使用滤镜，AVFrame 也会过一遍 空滤镜做下样子。</div>

音频流的 滤镜是通过 `configure_audio_filters()` 函数来创建的，因为 `ffplay` 为了代码的通用性，即便命令行参数不使用滤镜，`AVFrame` 也会过一遍 空滤镜做下样子。

 `configure_audio_filters()` 函数的流程图如下：

![1-1](configure_audio_filters\1-1.jpg)

`configure_audio_filters()` 函数的定义如下：

```
static int configure_audio_filters(VideoState *is, const char *afilters, int force_output_format){....}
```

下面讲解一下这个函数的参数。

`VideoState *is` ，是 ffplay 播放器的全局管理器。

`char *afilters`，是滤镜字符串，例如 下面的命令：

```
ffplay -af "atempo=2.0" -i juren-5s.mp4
```

`"atempo=2.0"` 这个字符串就会赋值给 `afilters` 。

`int force_output_format` ，代表是否强制把 `buffersink` 出口滤镜的音频帧采样等信息 设置为 跟 `is->audio_tgt` 一样。

之前说过 `is->audio_tgt` 是音响硬件设备打开的信息。`is->audio_tgt` 是**最终**要传递给 SDL 的音频格式。所有的采样率，声道数等等最后都要转成 `is->audio_tgt` 。

------

下面来分析一下`configure_audio_filters()` 函数里面的重点代码，如下：

![1-2](configure_audio_filters\1-2.png)

这个函数一开始就定义了 一些只有 2 个元素的数组，这其实是 ffmpeg 项目传递参数的方式，传递一个数组进去函数，主要有两种方式。

**1，**传递数组的大小。就是有多少个元素。

**2，**传递数组的结尾，只要读到结尾元素 (-1)，就算结束了。

ffmpeg 大部分函数采用的是第二种方式。

------

然后他会调 `avfilter_graph_free()` 释放**滤镜容器**（FilterGraph），有些同学可能会疑惑，`is->agraph` 一开始不是 `NULL` 吗？ 为什么需要释放？

`is->agraph` 一开始确实是 NULL，但是 `configure_audio_filters()` 这个函数可能会调用**第二次**，第二次的时候 `is->agraph` 就不是 NULL了。

`configure_audio_filters()` **第一次调用**是在  `stream_component_open()` 里面，如下：

![1-3](configure_audio_filters\1-3.png)

**第二次调用**是在 `audio_thread()` 里面，如下：

![1-4](configure_audio_filters\1-4.png)

第二次调用 `configure_audio_filters()` 是因为实际解码出来的 AVFrame 的采样率，声道等，跟容器里面记录的不一致，之前 `is->audio_filter_src` 是直接从容器，封装层取的数据。封装层记录的音频采样率等，**可能是错的**，需要以实际解码出来的 `AVFrame` 为准。

而且，注意，第二次的时候，`force_output_format` 参数会置为 1，这样会强制 `buffersink` 出口滤镜的采样信息等 设置为 `is->audio_tgt` 一样。

其实`configure_audio_filters()` **必然会调第二次的**，因为 `is->auddec.pkt_serial != last_serial`  这个条件肯定是真。

------

接着就是设置 滤镜使用的线程数量，0 为自动选择线程数量，如下：

```
is->agraph->nb_threads = filter_nbthreads;
```

------

第三个重点是，设置重采样选项（aresample_swr_opts），如下：

![1-5](configure_audio_filters\1-5.png)

什么样的命令行参数才是重采样选项的，在 `libswresample/options.c` 里面可以找到，如下：

![1-6](configure_audio_filters\1-6.png)

举个例子，如下：

```
ffpaly -ich 1 -i juren-5s.mp4
```

`ich 1` 就会被解析拷贝进去 `ffplay.c` 里面的 `swr_opts` 变量里面。

这里还用到了一个新的函数 `av_opt_set()`，这个函数其实不只可以设置**滤镜的属性字段**，还可以设置大多数**数据结构**的属性字段，例如解码器，封装器 等等，只要内部有 `AVClass` 的数据结构，都能用 `av_opt_set()` 来设置属性，详情请阅读《[AVOptions详解](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avoptions.html)》

------

接下来的重点是设置入口跟出口滤镜，如下：

![1-7](configure_audio_filters\1-7.png)

**出口滤镜**还设置了 `sample_fmts` 为 `AV_SAMPLE_FMT_S16`，这是 `ffpaly` 播放器自己的特性，就是说无论MP4文件里面的音频格式是怎样的，他都会转成 `AV_SAMPLE_FMT_S16` 格式丢给 SDL 播放，而且它在用 [SDL_OpenAudioDevice](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_open.html) 打开音频设备的时候，就是用的 S16 格式，**这是写死的**。

------

`force_output_format` 的逻辑主要是 强制 `buffersink` 出口滤镜的采样信息等 设置为跟 `is->audio_tgt` 一样。`audio_tgt` 是 SDL 接受音频帧的最终格式。

第一次调用 `configure_audio_filters()` 函数，`force_output_format` 为 0，不会跑进去这块逻辑。

------

最后就是调 `configure_filtergraph()` 函数来**链接入口跟出口滤镜**，同时创建滤镜容器（FilterGraph），如下：

![1-8](configure_audio_filters\1-8.png)

上图最重要的是，入口滤镜 跟 出口滤镜 被赋值到全局管理器 `is` 了。后面只要把解码器输出的 AVFrame 往入口滤镜丢，然后往出口滤镜读就行了。

------

`configure_filtergraph()` 函数的内部逻辑比较简单，请自行研究，不熟悉滤镜各个函数的，可以看[《FFmpeg的scale滤镜介绍》](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/scale.html)等滤镜文章。
