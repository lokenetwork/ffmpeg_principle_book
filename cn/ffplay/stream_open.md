# stream_open函数分析—ffplay.c源码分析

<div id="meta-description---">ffplay 的 stream_open 函数分析，以及基础的数据结构</div>

在讲 **stream_open()** 函数之前，需要先了解 `stream_open()` 里面使用到的一些基本的数据结构。如下：

第一个数据结构是 `struct VideoState` ，`VideoState` 可以说是播放器的**全局管理器**。字段非常多，时钟，队列，解码器，各种状态 都放在 `VideoState` 里面。

但是本文不会把  `VideoState` 的所有字段都讲一遍，只会讲  `stream_open()` 函数用到的字段，如下是精简过的字段，顺序也经过调整，方便阅读。：

```
typedef struct VideoState {
    int last_video_stream, last_audio_stream, last_subtitle_stream;
	char* filename;
	AVInputFormat* iformat;
	int width, height, xleft, ytop;
    FrameQueue pictq;
    FrameQueue sampq;
	PacketQueue videoq;
	PacketQueue audioq;
	SDL_cond *continue_read_thread;
	SDL_Thread *read_tid;
    Clock audclk;
    Clock vidclk;
    Clock extclk;    
	int audio_clock_serial;
	int audio_volume;
    int muted;
	int av_sync_type;
} VideoState;
```

**1，**`int last_video_stream `  代表最后一个视频流，如果你的音视频文件 里面有多个视频流，`last_video_stream `  就代表最后一个视频流。另外两个 `last_audio_stream`, `last_subtitle_stream` 一样代表最后一个。

**2，**`char* filename` 存储的是打开的 音视频文件名，或者是网络地址url。

**3，**`AVInputFormat* iformat`，容器格式，`ffplay` 默认是根据 `filename` 的后缀来确定容器格式，但是你也可以指定按某种容器格式来解析文件。命令如下：

```
ffplay -i juren-5s.mp4 -f flv
```

通过命令行参数指定的 `-f flv`  就会被存储到 `AVInputFormat* iformat`，当然不是存的字符，有一个根据字符串找到 `AVInputFormat` 的过程。

**4，**`int width, height, xleft, ytop;`，分别代表播放器窗口的 **宽高** 跟 **位置**。位置通过 `xleft` 跟 `ytop` 来定位的。

**5，**`FrameQueue pictq`， `FrameQueue sampq`，视频跟音频的 `AVFrame` 队列。

**6，**`PacketQueue videoq`，`PacketQueue audioq`，视频跟音频的 `AVPacket` 队列。

**7，**`SDL_cond *continue_read_thread`，这是一个 SDL 的**条件变量**，用于线程间通信的。`read_thread()` 线程在以下两种情况会进入休眠 10ms。

**第一种情况**：`PacketQueue` 队列满了，无法再塞数据进去。

**第二种情况**：超过最小缓存size。

如果在 10ms 内，`PacketQueue` 队列全部被消耗完毕，`audio_thread()` 或者 `video_thread()` 线程 没有 `AVPakcet` 能读了，就需要尽快唤醒 `read_thread()` 线程。

还有，如果进行了 `seek` 操作，也需要快速把 `read_thread()` 线程 从休眠中唤醒。

所以 `SDL_cond *continue_read_thread` 条件变量，主要用于 `read_thread` 跟 `audio_thread` ，`video_thread` 线程进行通信的。

**8，**`SDL_Thread *read_tid;`，`read_thread` 的线程ID。

C++14 标准库有**跨平台**的线程库，但是 C语言 是没有跨平台的线程库，所以 `ffplay` 取巧了，使用了 SDL 库的线程跟条件变量，SDL 是跨平台的。

**9，**`Clock audclk;`，音频时钟，记录音频流的目前的**播放时刻** 。

**10，**`Clock vidclk;`，视频时钟，记录视频流的目前的**播放时刻** 。

**11，**`Clock extclk;`，外部时钟，取第一帧 音频 或 视频的 pts 作为 起始时间，然后随着物理时间的消逝增长，所以是物理时间的当前时刻。到底是以音频的第一帧，还是视频的第一帧？取决于 `av_read_frame()` 函数第一次读到的是音频还是视频。

**12，**`int audio_clock_serial;`，这个字段比较独特，只有音频有，视频没有，没有一个 `video_clock_serial` 字段。

`audio_clock_serial` 只是一个用做临时用途的变量，实际上存储的就是 `AVFrame` 的 `serial` 字段。不用特别关注。而视频直接用的  `AVFrame` 的 `serial`。

**13，**`int audio_volume`，播放器的声音大小。

**14，**`int muted`，是否静音，**C语言C99标准是没有 bool 类型的**，都用 `int` 代替。

**15，**`int av_sync_type`，音视频同步方式，有 3 种同步方式，以音频时钟为准，以视频时钟为准，以外部时钟为准。默认方式是以音频时钟为准。

------

上面的数据结构，有一些字段我会说得比较简洁，因为现在只需你对这些字段有个简单的了解，后面文章会具体详细用到这些字段的场景。

由于 `FrameQueue` 跟 `PacketQueue` 这两个数据结构非常重要，所以放一整图片方便理解。

![1-2](stream_open\1-2.jpg)

`FrameQueue` 里面的 `queue` 是一个数组，16 在代码里是一个宏，那个宏通常等于 16。`PacketQueue` 里面的 `pkt_list` 是一个 `AVFifoBuffer`，推荐阅读[《FifoBuffer函数库详解》](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avfifobuffer.html)。

`FrameQueue` 跟 `PakcetQueue` 是通过一个 `pktq` 指针来关联的。这两个队列都有自己的 锁 `mutex` 跟 条件变量 `cond`。操作这两个队列都需要加锁操作的。



------

下面开始分析  `stream_open()` 函数，流程图，代码如下：

![1-3](stream_open\1-3.jpg)

![1-4](stream_open\1-4.png)

![1-5](stream_open\1-5.png)

从上面的图可以看出来，  `stream_open()` 函数的内部实现非常的简单。无非就是 初始化 队列，初始化时钟，然后创建一个 `read_thread()` 线程去跑。

其中 `frame_queue_init()` 的内部实现也比较简单，不过有几个重点，如下：

![1-6](stream_open\1-6.png)

这个 `!!` 操作没有什么特别，实际上就是把大于 1 的数字转成 1。如果 `keep_last` 等于 5，取反两次之后，就会变成 1 了。`keep_last `字段的作用 请看[《FrameQueue队列分析》](https://ffmpeg.xianwaizhiyin.net/ffplay/frame_queue.html)。

------

`packet_queue_init()` 函数里面也有一句代码需要注意。

```
q->abort_request = 1;
```

`abort_request` 字段如果置为 1， `audio_thread()` 跟 `video_thread()` 解码线程就会退出。所以在创建解码线程 之前，`ffplay` 会把 `abort_request` 置为 0 ，如下：

![1-7](stream_open\1-7.png)

------

至此，`stream_open()` 函数已经讲解完毕，**现在逻辑流程已经流转到新的线程 `read_thread()` 函数里面了**，下面文章继续分析 `read_thread()` 函数的内部原理。
