# video_thread视频解码线程分析—ffplay.c源码分析

<div id="meta-description---">video_thread 线程主要是负责 解码 PacketQueue 队列里面的 AVPacket 的，解码出来 AVFrame，然后丢给入口滤镜，再从出口滤镜把 AVFrame 读出来，再插入 FrameQueue 队列。</div>

之前在 `stream_component_open()` 里面的 `decode_start()` 函数开启了 `video_thread` 线程，如下：

![0-1](video_thread\0-1.png)

`video_thread` 线程主要是负责 **解码** `PacketQueue` 队列里面的 `AVPacket` 的，解码出来 `AVFrame`，然后丢给**入口滤镜**，再从**出口滤镜**把 `AVFrame` 读出来，再插入 `FrameQueue` 队列。流程图如下：

![1-1](video_thread\1-1.jpg)

`video_thread()` 函数里面有几个 `CONFIG_AVFILTER` 的宏判断，这是判断编译的时候是否启用滤镜模块。默认都是启用滤镜模块的。

下面来分析一下 `video_thread()` 函数的重点逻辑，如下：

![1-2](video_thread\1-2.png)

 `video_thread()` 函数里面比较重要的局部变量如下：

1，`AVFilterGraph *graph`，滤镜容器

2，`AVFilterContext *filt_in`，入口滤镜指针，指向滤镜容器的输入

3，`AVFilterContext *filt_out`，出口滤镜指针，指向滤镜容器的输出

4，`int last_w` ，上一次解码出来的 `AVFrame` 的宽度，初始值为 0

5，`int last_h` ，上一次解码出来的 `AVFrame` 的高度，初始值为 0

6，`enum AVPixelFormat last_format` ，上一次解码出来的 `AVFrame` 的像素格式，初始值为 -2

7，`int last_serial`，上一次解码出来的 `AVFrame` 的序列号，初始值为 -1

8，`int last_vfilter_idx `，上一次使用的视频滤镜的索引，`ffplay` 播放器的命令行是可以指定多个视频滤镜，然后按 `w` 键切换查看效果的

声明初始化完一些局部变量之后，`video_thread()` 线程就会进入 `for` 死循环不断处理任务。

`get_video_frame()` 函数主要是从解码器读取 `AVFrame`，里面有一个**视频同步**的逻辑，同步的逻辑稍微复杂，推荐阅读《[FFplay视频同步分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_sync.html)》

------

需要注意的是，`last_w` 一开始是赋值为 0 的，所以必然不等于解码出来的 `frame->width`，所以一开始肯定是会调进入那个 `if` 判断，然后调 [configure_video_filters()](https://ffmpeg.xianwaizhiyin.net/ffplay/video_filter.html) 函数创建滤镜。

总结一下，释放旧滤镜，重新创建新的滤镜有3种情况：

**1，**后面解码出来的 `AVFrame` 如果跟上一个 `AVFrame` 的宽高或者格式不一致。

**2，**按了 `w` 键，`last_vfilter_idx != is->vfilter_idx`。`ffplay` 播放器的命令行是可以指定多个视频滤镜，然后按 `w` 键切换查看效果的，推荐阅读《[FFplay视频滤镜分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_filter.html)》。

**3，**进行了快进快退操作，因为快进快退会导致 `is->viddec.pkt_serial` 递增。详情请阅读《[FFplay序列号分析](https://ffmpeg.xianwaizhiyin.net/ffplay/serial.html)》。我也不知道为什么序列号变了要重建滤镜。

这3种情况，`ffplay` 都会处理，只要解码出来的 `AVFrame` 跟之前的格式不一致，都会重建滤镜，然后更新 `last_xxx` 变量，这样滤镜处理才不会出错。

由于每次读取**出口滤镜**的数据，都会用 `while` 循环把缓存刷完，不会留数据在滤镜容器里面，所以重建滤镜不会导致数据丢失。

------

`video_thread` 线程的逻辑比较简单，复杂的地方都封装在它调用的**子函数**里面，所以本文简单讲解一下，`video_thread()` 里面调用的各个函数的作用。

**1，**`get_video_frame()`，实际上就是对 `decoder_decode_frame()` 函数进行了封装，加入了**视频同步逻辑**。返回值如下：

- 返回 1，获取到 `AVFrame` 。
- 返回 0 ，获取不到 `AVFrame` 。有3种情况会获取不到 AVFrame，一是MP4文件播放完毕，二是解码速度太慢无数据可读，**三是视频比音频播放慢了导致丢帧**。
- 返回 -1，代表 `PacketQueue` 队列关闭了（`abort_request`）。返回 `-1`  会导致 `video_thread()` 线程用 `goto the_end` 跳出 `for(;;)` 循环，跳出循环之后，`video_thread` 线程就会自己结束了。返回 `-1` 通常是因为关闭了 `ffplay` 播放器。

更详细的分析请阅读《[FFplay视频同步分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_sync.html)》

---

**2，**`configure_video_filters()`，创建视频滤镜函数，推荐阅读《[FFplay视频滤镜分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_filter.html)》。

**3，**`av_buffersrc_add_frame()`，往入口滤镜发送 `AVFrame`。

**4，**`av_buffersink_get_frame_flags()`，从出口滤镜读取 `AVFrame`。

滤镜相关的函数推荐阅读 **FFmpeg实战之路** 一章的 《[FFmpeg的scale滤镜介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/scale.html)》

---

**5，**`queue_picture()`，此函数可能会阻塞。只是对 `frame_queue_peek_writable()` 跟 `frame_queue_push()` 两个函数进行了封装。

在 `audio_thread()` 音频线程里面是用 `frame_queue_peek_writable()` 跟 `frame_queue_push()` 两个函数来插入 `FrameQueue` 队列的。

在 `video_thread()` 视频线程里面是用 `queue_picture()` 函数来插入 `FrameQueue` 队列的。

音频解码线程 跟 视频解码线程，有很多类似的地方，跟 `FrameQueue` 队列相关的函数都在 《[FrameQueue队列分析](https://ffmpeg.xianwaizhiyin.net/ffplay/frame_queue.html)》一文中。
