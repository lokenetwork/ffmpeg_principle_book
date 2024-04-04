# FFplay序列号分析—ffplay.c源码分析

<div id="meta-description---">序列号 主要是给 快进快退 这个功能准备的。如果不能快进快退，那其实就不需要序列号。只要解复用线程不断读取 AVPacket 放进去 PacketQueue 队列，解码线程不断从 PacketQueue 取数据来解码放进去 FrameQueue，最后有播放线程来取 FrameQueue 的数据取播放就行。</div>

在之前的几篇文章里面，都零零散散提及过**序列号**这个概念。但是 序列号 这个概念对于 `FFplay` 播放器 非常重要的，很多代码都跟序列号有关。所以单独写一篇文章介绍序列号。

序列号 主要是给 **快进快退** 这个功能准备的。如果不能快进快退，那其实就不需要序列号。只要解复用线程不断读取 `AVPacket` 放进去 `PacketQueue` 队列，解码线程不断从 `PacketQueue` 取数据来解码放进去 `FrameQueue`，最后有播放线程来取 `FrameQueue` 的数据取播放就行。

整体的流程图如下：

![1-1](serial\1-1.jpg)

`PacketQueue` 跟 `FrameQueue` 都是**缓存**队列，都是先提取准备好要用的数据。但是就是因为加了 快进快退 这个功能，一旦跳转到别的时间点播放，之前提前准备好的数据是不是就不能用了？

所以就需要一个序列号来判断，之前缓存的数据是不是最新的，**所以序列号可以看成是版本号一样的东西**。

------

**FFplay** 播放器主要有 4 个序列号字段，分别如下：

**1，**`struct PackeQueue` 的 `serial` ，这是队列本身的序列号。可以看成是最新的序列号的值。

**2，** `struct MyAVPacketList` 的 `serial` ，队列里面 `AVPacket` 的序列号。

**3，**`struct Decoder` 的 `pkt_serial`，记录的是解码器上一次用来解码的 `AVPacket` 的序列号

**4，**`struct Frame` 的 `serial`，队列里面 `AVFrame` 的序列号。

首先，`MyAVPacketList` 的 `serial` 来源与  `PackeQueue` 的 `serial`，在 `packet_queue_put_private()` 函数里面可以看到：

![1-2](serial\1-2.png)

`Frame` 的 `serial` 是来源于 `Decoder` 的 `pkt_serial`，如下：

![1-3](serial\1-3.png)

而 `Decoder` 结构里面的 `pkt_serial` 又是来自哪里的呢？

**答：**是在 `decoder_decode_frame()` 里面进行赋值的。记录的是最近一次输入解码器的  `AVPacket` 的序列号，如下：

![1-4](serial\1-4.png)

因此，可以说，`Frame` 的 `serial` 是**间接来源**于 `AVPacket` 的 `serial` 的。

------

现在已经弄懂了 4 个序列号赋值的地方，那这 4 个序列号字段用来哪些代码里面呢？

**答：只要跟缓存有关的逻辑，都会判断序列号。**

------

因为序列号是为了 快进快退 功能准备的，所以**先简单讲解**一下 快进快退的逻辑，更详细的分析请看《[FFplay跳转时间点播放](https://ffmpeg.xianwaizhiyin.net/ffplay/jump.html)》。

当你按下上下左右其中一个键的时候，就会产生一个 seek 操作，这是调 `stream_seek()` 函数实现的，如下：

![1-5](serial\1-5.png)

 `stream_seek()` 函数的代码如下：

```
/* seek in the stream */
static void stream_seek(VideoState *is, int64_t pos, int64_t rel, int seek_by_bytes)
{
    if (!is->seek_req) {
        is->seek_pos = pos;
        is->seek_rel = rel;
        is->seek_flags &= ~AVSEEK_FLAG_BYTE;
        if (seek_by_bytes)
            is->seek_flags |= AVSEEK_FLAG_BYTE;
        is->seek_req = 1;
        SDL_CondSignal(is->continue_read_thread);
    }
}
```

上面的代码重点是设置 了 `is->seek_req` 标记 以及 要跳到到哪些位置播放。然后就会唤醒 `read_thread()` 线程。 `read_thread()` 线程看到 `is->seek_req` 这个标记，就会开始进行 `seek` 操作。如下：

所以，真正进行 `seek` 操作的地方是在  `read_thread()` 解复用线程里面，而不是 `main` 线程。

![1-6](serial\1-6.png)

从上图可以看出来，用 `avformat_seek_file()` 进行 `seek` 操作之后，就会 调 `packet_queue_flush()` 函数来刷新 序列号，来达到丢弃之前的无效缓存的效果。

`packet_queue_flush()` 是一个非常重点的函数，代码如下：

![1-7](serial\1-7.png)

可以看到，一开始就清空了 `PacketQueue` 队列里面的数据，然后把序列号 `+1`。

我个人有个疑问，既然 `+1` 之前已经把 数据清空，那 解码线程 从 `PakcetQueue` 取数据的时候根本不需要判断序列号。因为已经清空了，解码线程不可能取到旧的 `AVPacket`。

先不管这个，反正 `PacketQueue` 的 `serial` 序列号刷新了。

------

`PacketQueue` 队列本身的 `serial` 序列号 `+1` 刷新，**会导致一系列的连环反应**。

首先是**后面** 解复用线程往  `PacketQueue` 队列 加进去的 `MyAVPacketList` 的 `serial` 也会是最新的。但是，解码管理器（`struct Decoder`）里面的 `pkt_serial` 还是旧值。

之前说过，`struct Decoder` 的 `pkt_serial` 记录的是上一次解码的 `AVPacket` 的序列号。

这就会导致，直接放弃解码器里面的缓存，用新的 `AVPacket` 进行解码，如下：

![1-8](serial\1-8.png)

同样会导致 解码器的缓存被刷新，如下：

![1-9](serial\1-9.png)

上图的逻辑是，只要本次解码的 `AVPacket` 的序列号跟上一次用的 `AVPacket` 的序列号不一样，**立即刷新解码器缓存**。

------

还有一个被连环影响的地方就是 读取 `FrameQueue` 的逻辑，`FrameQueue` 跟 `PakcetQueue` 不太一样，`FrameQueue` 本身是没有序列号的，只是它队列里面的 `struct Frame` 有 序列号，所以也会受到影响，如下：

![2-1](serial\2-1.png)

可以看到，音频播放线程取数据的时候，会判断 `Frame` 的序列号是不是最新的，不是就立即丢弃。

------

序列号应用的场景还有一个，那就是时钟（Clock），如下：

```
typedef struct Clock {
   	...
    int serial;           /* clock is based on a packet with this serial */
    ...
    int *queue_serial;    /* pointer to the current packet queue serial, used for obsolete clock detection */
} Clock;
```

这个场景是音视频同步的时候用的，有点复杂。后面讲解音视频同步的时候会再分析 `Clock` 里面的序列号。

