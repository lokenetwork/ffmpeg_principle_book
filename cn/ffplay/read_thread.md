# read_thread解复用线程分析—ffplay.c源码分析

<div id="meta-description---">ffplay 的 read_thread线程分析</div>

**read_thread()** 线程的主要作用从 MP4 里面读取 `AVPacket`，然后丢进去 `PacketQueue` 队列。所以需要先学习一下 `strcut PacketQueue` 跟  `struct MyAVPacketList` 数据结构。如下：

```
typedef struct MyAVPacketList {
    AVPacket *pkt;
    int serial;
} MyAVPacketList;
```

```
typedef struct PacketQueue {
    AVFifoBuffer *pkt_list; //存储的是 MyAVPacketList
    int nb_packets;
    int size;
    int64_t duration;
    int abort_request;
    int serial;
    SDL_mutex *mutex;
    SDL_cond *cond;
} PacketQueue;
```

**1，**`AVFifoBuffer *pkt_list` ，`AVFifoBuffer` 是一个 [circular buffer FIFO](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avfifobuffer.html)，一个环形的先进先出的缓存实现。里面存储的是 `struct MyAVPacketList` 结构的数据。

**2，**`int nb_packets;`，代表队列里面有多少个 `AVPacket`。

**3，**`int size;` ，队列缓存的数据大小 ，算法是**所有的** `AVPacket` 本身的大小加上 `AVPacket->size` 。

**4，**`int64_t duration`，队列的时长，通过累加 队列里所有的 `AVPacket->duration` 得到。

**5，**`abort_request`，代表队列终止请求，变成 1 会导致 `audio_thread` 跟 `video_thread` 退出。

**6，**`int serial`，队列的序号，每次跳转播放时间点 ，`serial` 就会 `+1`。另一个数据结构 `MyAVPacketList`  里面也有一个 `serial` 字段。

两个 `serial` 通过比较匹配来丢弃无效的缓存帧，什么情况会导致队列的缓存帧无效？跳转播放时间点的时候。

例如此时此刻，`PacketQueue` 队列里面缓存了 8 个帧，但是这 8 个帧都 **第30分钟** 才开始播放的，如果你通过 ➔ 按键前进到 **第35分钟** 的位置播放，那队列的 8 个缓存帧就无效了，需要丢弃。

由于每次跳转播放时间点， `PacketQueue::serial` 都会 `+1` ，而 `MyAVPacketList::serial`  的值还是原来的，两个 serial 不一样，就会丢弃帧。

**7，**`SDL_mutex *mutex` ，SDL 互斥锁，主要用于修改队列的时候加锁。

**8，**`SDL_cond *cond`，SDL 条件变量，用于 `read_thread()` 线程 跟 `audio_thread()` ，`video_thread()` 线程 进行通信的。

------

在 `ffplay -i juren-5s.mp4` 的场景下，`read_thread` 线程的流程图如下：

![1-1](read_thread\1-1.jpg)

`read_thread()` 线程里面的逻辑相对比较复杂，重点也挺多。首先讲解一下 `st_index[]` 这个数组变量的含义，如下：

![1-2](read_thread\1-2.png)

`st_index[]` 这个数组用的宏是 `AVMEDIA_TYPE_NB`，也就是这个数组涵盖了各种数据流，音频，视频，字幕，附件流等等。因为一个MP4里面可能会有多个视频流。

例如 第 5，第 6 个流都是视频流。这时候 `st_index[AVMEDIA_TYPE_VIDEO]` 保存的可能就是 5 或者 6 ，代表要播放哪个视频流，其他数据流类推。

默认 `st_index[]`  数组的值是通过 `av_find_best_stream()` 确定的，是通过 `bit_rate` 最大比特率，`codec_info_nb_frames` 等参数找出 **最好**的那个音频流 或者 视频流。



------

第二个重点是 `interrupt_callback` 这个操作，指定了中断回调函数。

![1-3](read_thread\1-3.png)

`decode_interrupt_cb()` 函数实现如下：

```
static int decode_interrupt_cb(void *ctx)
{
    VideoState *is = ctx;
    return is->abort_request;
}
```

首先，`is->abort_request` 这个变量控制着整个播放器要不要停止播放，然后退出。

在播放本地文件的时候，`interrupt_callback` 回调函数的作用不是特别明显，因为本地读取MP4， `av_read_frame()` 会非常快返回。

但是如果在播放网络流的时候，网络卡顿，`av_read_frame()` 可能要 8 秒才能返回，这时候如果想关闭播放器，就需要 `av_read_frame()` 尽快地返回，不要再阻塞了。这时候，就需要 `interrupt_callback` 了，因为在 8 秒 内，`av_read_frame()` 内部也会定时执行 `interrupt_callback()`，只要 `interrupt_callback()` 返回 1，`av_read_frame()` 就会不再阻塞，立即返回。

提醒：播放网络流的时候，`avformat_find_stream_info()` 可能会跟 `av_read_frame()` 一样阻塞很久。

------

`read_thread()` 线程的**第三个重点**是 `avformat_open_input()` 函数的使用，在《[FFmpeg打开输入文件](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/input.html)》一文中，已经讲过这个函数的使用了，但是没有讲最后一个参数的用法。

![1-3-2](read_thread\1-3-2.png)

最后的参数 `format_opts` 是一个 `AVDictionary` （字典）。注意，如果 `avformat_open_input` 函数内部使用了字典的某个选项，**就会把这个选项从字典剔除**。

所以可以看到，后面判断了还有哪些 `option` 没使用，这些无法使用的 `option` （选项），通常是因为命令行参数写错了。

MP4，FLV，TS，等等容器格式，都有一些相同的 option，也有一些不同的 options。具体可以通过以下命令查看容器支持哪些 option ？

```
ffmpeg -h demuxer=mp4
```

提示：各种流媒体格式 也可以看成是 容器。

------

`read_thread()` 里面会处理 `seek` 操作，但是本文是讲解 `ffplay -i juren-5s.mp4`  **简单场景**下的逻辑的。

简单场景下，不会跑进去 `seek` 条件。 `seek` 操作可以后面再看这篇文章[《FFplay跳转时间点播放》](https://ffmpeg.xianwaizhiyin.net/ffplay/jump.html)

------

`read_thread()` 线程的第**四个重点**是 `AVRational sar` 变量的应用，如下：

![1-4](read_thread\1-4.png)

`sar` 这个值是不太容易理解的，我刚开始也被这个 `sar` 搞懵。我之前以为 `sar` 等于 `width/height` （宽高比） ，后来发现不是宽高比。

其实 `sar` 是以前的显示设备设计的历史遗留问题，不用过多关注，只需要知道，显示的时候用 `sar` 这个比例拉伸 `width` 跟 `height` 作为显示窗口，图像播放就不会扭曲了。`sar` 在大部分情况都是 `1:1`。

推荐阅读，[ffmpeg解析出的视频参数PAR，DAR，SAR的意义](https://blog.csdn.net/zhizhuodewo6/article/details/105247557) 跟 [theory-videoaspectratios](https://www.animemusicvideos.org/guides/avtech3/theory-videoaspectratios.html)

------

接下来来到 `read_thread()` 线程里最重要的重点，`stream_component_open()` 函数的调用，`audio_thread()`，`video_thread()` 等解码线程就是从 `stream_component_open()` 里 创建出来的。推荐阅读《[stream_component_open函数分析](https://ffmpeg.xianwaizhiyin.net/ffplay/stream_component_open.html)》

------

上面所有代码干的活，主要是找出最好的音视频流，设置回调，各种初始化，打开容器实例。

现在到了  `read_thread()` 线程的**主要任务**，那就是进入 `for (;;) {...}` 死循环不断 从 容器实例 读取 `AVPacket` ，然后丢进去对应的 `PacketQueue` 队列

`for` 循环里面也有一些重点，如下：

![1-5](read_thread\1-5.png)

对于播放本地文件，`av_read_pause()` 函数其实是没有作用的。`av_read_pause()` 只对网络流播放有效，有些流媒体协议支持暂停操作，暂停了，服务器就不会再往 `ffplay` 推送数据，如果想重新推数据，需要调用 `av_read_play()`

------

`for` 循环里面的第二个重点是 判断 队列缓存中的 `AVPacket` 是否够用，够用就会休眠 10ms。如下：

![1-6](read_thread\1-6.png)

在播放本地文件的时候，`infinite_buffer` 总是 0，所以不用管它。

可以看到，判断 `AVPacket` 是否够用，就是根据 size 来判断，还有 `stream_has_enough_packets()` 函数，实现如下：

```
static int stream_has_enough_packets(AVStream *st, int stream_id, PacketQueue *queue) {
    return stream_id < 0 ||
           queue->abort_request ||
           (st->disposition & AV_DISPOSITION_ATTACHED_PIC) ||
           queue->nb_packets > MIN_FRAMES && (!queue->duration || av_q2d(st->time_base) * queue->duration > 1.0);
}
```

`stream_has_enough_packets()` **主要就是**确认 队列至少有 `MIN_FRAMES` 个帧，而且所有帧的播放时长加起来大于 1 秒钟。

------

当 队列缓存中的 `AVPacket` 未满的时候，就会直接去读磁盘数据，把 `AVPacket` 读出来，但是也不是读出来就会立即丢进去 `PacketQueue` 队列，而是会判断一下`AVPacket ` 是否在期待的播放时间范围内。如下：

![1-7](read_thread\1-7.png)

可以看到 定义了 一个 变量 `pkt_in_play_range` 来确定是否在播放时间范围内。播放时间范围这个概念是这样的。如果下面这样播放一个视频：

```
ffplay -i juren-5s.mp4
```

因为 `juren-5s.mp4` 是一个 5 秒的视频，而且命令行没有指定 `-t`，所以这时候 播放时间范围 就是 0 ~ 5 秒。只要读出来的 `AVPacket` 的 `pts` 在 0 ~ 5秒范围内，`pkt_in_play_range` 变量就为真。因此所有读出来的 AVPacket 都是符合播放时间范围的。

但是如果加了 `-t` 参数，如下：

```
ffplay -t 2 -i juren-5s.mp4
```

上面的的命令是 只播放 2秒视频，也就是 播放时间范围 变成了  0 ~ 2 秒，如果读出来的 `AVPacket` 的 `pts` 大于 2 秒，就会被丢弃。

这里就有一个有趣的事情，当视频播放到 第二秒的时候，虽然画面停止了，但是 `read_thread()` 还是会一直读数据，但由于不符合播放时间范围，会一直丢弃。直到读到文件结尾，返回 `AVERROR_EOF` 才会停下来休眠一小段时间。

------

读出来的 `AVPacket`  符合播放时间之后，就会 用 `packet_queue_put()` 丢进去  `PacketQueue` 队列。

可以看到，音频，视频流，是有各自的  `PacketQueue` 队列的，`is->audioq`  跟 `is->videoq`。

------

**FFplay** 播放器的逻辑流转，目前就转到 `for (;;) {...}` 循环里面不断读取 `AVPacket` 数据。

`read_thread()` 线程函数最后的 `fail:` 标签代码，是播放器退出之后的清理逻辑，这个目前不需要理会，可以后续再看《FFplay退出处理》。

------

`read_thread()` 线程里面有几个逻辑，本文是 **刻意忽略** 或者 **一笔带过** 了的，这些也是可以后续再看的，分别是：

**1，**`wanted_stream_spec[]` 数组的作用，本文中，这个数组全是 `-1`，所以忽略了。推荐阅读 《FFplay指定数据流播放》

**2，** `av_format_inject_global_side_data()` ，推荐阅读《av_format_inject_global_side_data函数详解》

**3，**`seek` 操作，推荐阅读[《FFplay跳转时间点播放》](https://ffmpeg.xianwaizhiyin.net/ffplay/jump.html)

