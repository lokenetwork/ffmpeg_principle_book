# FFmpeg的API库介绍—FFmpeg API教程

FFmpeg 的 API 库一共有 8 个，如下图：

![intro-1-1](intro\intro-1-1.png)



------

**1，swscale** ，视频数据 处理类库，例如提供了 sws_scale 函数来做 像素格式和分辨率的转换，还有一些处理视频的滤波函数。sw 应该是 SoftWare 的缩写。

**2，swresample** ，音频数据 处理类库，例如提供了 swr_convert 函数 来实现 音频数据的重采样。

**3，postproc**，视频**后处理**库，提供了一些视频相关的函数，但是有很多函数没有实现，或者实现了一半，如下：

![intro-1-2](intro\intro-1-2.png)

所以 libpostproc 应该是一个实验性的库，不用管。

------

**4，avutil**，a 代表 audio，v 代表 video，所以 avutil 是一个 与 音频，视频都相关的工具类。

avutil 这个库的主要函数分为以下几类。

- 流媒体相关的函数，例如 av_frame_get_pkt_duration，av_frame_get_pkt_pos。
- 内存管理函数，例如 av_malloc 跟 av_free 。
- 数学相关的函数，例如 av_mod_i，av_mul_q。
- FFmpeg 通用数据结构管理函数，例如 av_opt_find。
- 线程相关函数，例如 av_thread_message_queue_alloc。

可以看出， avutil 库是一个大杂烩，什么都有。

------

**5，avformat**，封装格式处理库，主要是解析 MP4，MP3，TS，FLV 等等封装格式，同时 FFmpeg 还自己创建了一些假的封装格式，例如 tee 封装格式，这些假的封装格式只是为了方便 实现 ffmpeg 命令行的功能。tee 就是为了实现 命令行多路输出的语法。

**6，avfilter**，音视频滤镜库。有非常多的滤镜可以选择，例如裁剪时间，加水印，画中画，音频倍速。

滤镜库其实有点杂，他不只是做一些特效。FFmpeg 把一些功能性的函数也会加进去 avfilter 库，例如 转换音频的采样率，声道 都是用 aformat 滤镜实现的。用 aformat 滤镜实现 重采样 会比 用 swresample 更简单。虽然内部也是调的 swresample 。

**7，avdevice**，设备处理类库，主要处理各种设备的输入信息，例如摄像头，麦克风，抓屏。

**8，avcodec**，编解码类库，avcodec 实际上是编解码管理器，他定义了一种通用的数据结构 来对接 各种编解码器，你可以把 很多外部的编解码 集成到 `avcodec` 类库。

------

至此 ，8 个库就介绍完了，`ffmpeg.exe` 实际上也是 调这 8个库的函数实现的。而 `ffmpeg.exe` 主要由一个 5 千行的 `ffmpeg.c` 文件实现的。

所以 怎么调这 8个库的函数，这些 API 库的使用方法其实就在  `ffmpeg.c` 里面。

`ffmpeg.exe` 命令行任何的功能，都能在 5 千行的 `ffmpeg.c` 找到实现的原理。

