# decoder_decode_frame解码函数分析—ffplay.c源码分析

<div id="meta-description---">decoder_decode_frame() 其实是一个通用的解码函数，可以解码 音频，视频，字幕的 AVPacket。不过本文主要侧重于分析音频流的解码，但其他的流也是类似的逻辑。</div>

`decoder_decode_frame()` 其实是一个通用的**解码函数**，可以解码 音频，视频，字幕的 `AVPacket`。不过本文主要侧重于分析**音频流的解码**，但其他的流也是类似的逻辑。

`decoder_decode_frame()` 函数的参数如下：

```
static int decoder_decode_frame(Decoder *d, AVFrame *frame, AVSubtitle *sub) 
```

**1，**`Decoder *d`，这个 `Decoder` 结构，我把它称为**解码管理实例**。由于 `Decoder` 结构里面有**解码器实例** 跟 `PacketQueue` 队列，所以只需要传递 `Decoder` 给 `decoder_decode_frame()` 函数就能进行解码了。如下：

```
typedef struct Decoder {
    ...
    PacketQueue *queue; // AVPacket 队列
    AVCodecContext *avctx; //解码器实例
	...
} Decoder
```

**2，** `AVFrame *frame` ，用来存储解码出来的音频或者视频的 `AVFrame`。

**3，** `AVSubtitle *sub` ，用来存储解码出来的字幕数据，字幕流的使用的数据结构不是 `AVFrame`，而是 `AVSubtitle ` 。

------

`decoder_decode_frame()` 函数的流程图如下：

![1-1](decoder_decode_frame\1-1.jpg)

`decoder_decode_frame()` 函数的**代码逻辑**跟我上面画的流程图的顺序有点不一样，但是整体的逻辑就是这么一个逻辑：**从 `PacketQueue` 队列拿数据去解码**。

------

下面来分析一下 `decoder_decode_frame()` 的代码，如下：

![1-2](decoder_decode_frame\1-2.png)

从上图可以看到，一开始就会用 `avcodec_receive_frame() `去解码器读数据，这里读者可能会有疑问，明明都还没往解码器发送 `AVPacket`， `avcodec_receive_frame() ` 函数怎么可能读取到 `AVFrame` 呢？

**答：**没错，就是读取不到，因为还没发 `AVPacket` 给解码器解码。所以，**首次**  `avcodec_receive_frame() ` 必然返回 `EAGAIN`，所以就会跳出这个 `do{}while{}` 循环。



------

跳出第一个 `do{}while{}` 循环之后，就会进入第二个  `do{}while{}` 循环，如下：

![1-3](decoder_decode_frame\1-3.png)

虽然代码比较少，但是**句句都是重点**。

**1，**`SDL_CondSignal(d->empty_queue_cond)`，首先，如果 `PacketQueue` 队列里面如果没有数据可读了，就需要唤醒 `read_thread()` 线程来读数据，之前说过， `empty_queue_cond` 实际上就是 `continue_read_thread`，这两个指针都指向同一个**条件变量**。

**2，**判断之前发送 `AVPacket` 给解码器是否失败了？如果失败，`d->packet_peding` 会是 1。如果上次失败了，`d->pkt `本身就是有值的，就不需要重队列里面拿数据，直接把 `d->pkt` 发送给解码器即可。

------

**3，**调用 `packet_queue_get()` 从 队列读取 `AVPacket`。简单讲解一下 `packet_queue_get()` 函数的**参数**，定义如下：

```
static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block, int *serial)
```

-  `int block` 是控制  `packet_queue_get()` 函数阻塞读取的，如果 `PacketQueue` 队列里面如果没有数据可读，可以一直阻塞等到 `read_thread()` 线程读到数据放进去队列 为止。
- `AVPacket *pkt`，用来放从队列读取到的 `AVPacket`。
- `int *serial`，读取到的 `AVPacket` 的序列号，这是一个返回值。传的是指针。

`PacketQueue` 队列是一个 [FIFO](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avfifobuffer.html) 的内存管理器，存的是 `MyAVPacketList`，如下：

```
typedef struct MyAVPacketList {
    AVPacket *pkt;
    int serial;
} MyAVPacketList;
```

也就是说，队列里的每一个 `AVPacket` 都有一个序列号的。

再回到  `decoder_decode_frame()` 调用  `packet_queue_get()` 时的传参，如下：

![1-4](decoder_decode_frame\1-4.png)

可以看到从队列取出来的 `AVPacket` 就放在 `d->pkt` 里面，而序列号就放在 `d->pkt_serial`。

FFplay播放器其实有 3 种序列号：

**1，** `MyAVPacketList` 的 `serial` ，队列里面的 `AVPacket` 的序列号。可以看成是临时值，旧值。

**2，**`PackeQueue` 的 `serial` ，这是队列本身的序列号。可以看成是最新的序列号的值。

**3，**`Frame` 的 `serial`，本文不需要关注这个，在《[FFplay序列号分析](https://ffmpeg.xianwaizhiyin.net/ffplay/serial.html)》一文会详细讲解。

`MyAVPacketList` 的 `serial` 就是用`PackeQueue` 队列的 `serial` 来赋值的 。如下代码：**只要不进行跳转播放，他们的序列号就是一样的**。

![1-5](decoder_decode_frame\1-5.png)

当快进快退，或者跳转播放时间点的时候，`PackeQueue` 队列的序列号就会 `+1`，而之前已经放进去队列的 `MyAVPacketList` 的序列号则**保持不变**。

举个例子，当前MP4已经播放到了 第20秒的时刻，此时 `PackeQueue` 队列缓存了 5 帧第 21秒的数据，此时，我快进30秒，跳转到第 50 秒的时刻播放。

由于跳转了，所以队列的序列号会 `+1`，变成了 2，而之前的 5 帧 `MyAVPacketList` 的序列号还是 1。两者就会不一样。

因为要开始播放第 50 秒的数据，所以 `PackeQueue` 队列之前缓存的 5 帧数据就不可用了。丢弃这 5 帧数据就是由下面的代码实现的。

![1-6](decoder_decode_frame\1-6.png)

注意看上面圈出来的代码，只有两者相等，才会 break 退出，**如果不相等**，就会直接 `av_packet_unref()` 释放，然后再进入 `while` 循环再从队列取`AVPacket`。

这样，就能把无效的 5 帧数据全部丢弃，直到从队列读取到序列号一致的 `AVPacket` 为止。

------

还有一个重点，就是当从队列取出来的 `AVPacket` 跟上一次取的 `AVPacket` 序列号不一样，就会刷新解码器的缓存。

序列号不一样，肯定是因为跳转了播放时间点，而解码器要按顺序解码的，如果不清空缓存，可能会导致马赛克。

![1-7](decoder_decode_frame\1-7.png)

------

假设现在已经从队列读取到 序列号跟队列一样的 `AVPacket`，就会把 `AVPacket` 发送给解码器，如下：

![1-8](decoder_decode_frame\1-8.png)

由于发送解码器可能会失败，所以 `ffplay` 做了一下处理，如果失败，就不 `unref`，直接标记一下 `d->packet_pending`，下次再继续发送。

------

此时，已经成功发送 `AVPacket` 给解码器了。这时候，`decoder_decode_frame()` 还没有结束，此时此刻还没跳出 最开始的 `for (;;) {}` 循环。

所以又会**重新进入**一开始的往解码器读 `AVFrame` 的逻辑，`decoder_decode_frame()`函数的逻辑可以说是**反着来的**，所以看起来有点奇怪。

现在回到一开始的逻辑，如下：

![1-9](decoder_decode_frame\1-9.png)

此时，`avcodec_receive_frame()` 仍然可能还是会返回 `EAGAIN`，因为不是往解码器发一个`AVPacket`，就一定有数据可读的。有些是 B 帧，还需要多一个P帧来解码。所以如果 `avcodec_receive_frame()` 返回 `EAGAIN`，就会从 `PacketQueue` 队列再拿出一个 `AVPacket` 往解码器丢。

现在我们假设 `avcodec_receive_frame()` 能读出数据了，可以看到，它赋值给了参数 `frame`，也就是第二个参数。

读到 `AVFrame` 之后，`decoder_decode_frame()` 就会直接 `return 1` 退出了。

注意，`decoder_decode_frame()` 只从解码器读取到一个 `AVFrame` 就返回了，如果解码器里面还有缓存的 `AVFrame`，下次就可以直接取，**而不用**再从队列拿 `AVPacket` 再发送给解码器。

**这就是为什么从解码器读 `AVFrame` 要加上这个 `if` 判断：**

```
if (d->queue->serial == d->pkt_serial) {
	...
}
```

因为已经跳转到别的时间播放了，**解码器的缓存是以前的时间点缓存的**。如果还继续取，窗口画面就有短暂的不准确。

------

最后做下总结，`decoder_decode_frame()` 函数的逻辑就是从解码器读取到 **一个** `AVFrame`，为了解码出**一个**`AVFrame`，它会从 `PacketQueue` 队列取 `AVPacekt` 发送给解码器，需要多少个就取多少个 `AVPacekt`，直至到能解码出一个 `AVFrame`。

`decoder_decode_frame()` 函数有 3 个返回值。

- 返回 1，获取到 `AVFrame` 。
- 返回 0 ，获取不到 `AVFrame` ，0 代表已经解码完MP4的所有`AVPacket`。这种情况一般是 `ffplay` 播放完了整个 MP4 文件，窗口画面停在最后一帧。但是由于你可以按 C 键重新循环播放，所以即便返回 0 也不能退出 `audio_thread` 线程。
- 返回 -1，代表 `PacketQueue` 队列关闭了（`abort_request`）。返回 `-1` 会导致 `audio_thread()` 函数用 `goto the_end` 跳出 `do{}whlle{}` 循环，跳出循环之后，`audio_thread` 线程就会自己结束了。返回 `-1` 通常是因为关闭了 `ffplay` 播放器。





