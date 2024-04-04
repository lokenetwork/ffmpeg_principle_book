# FrameQueue队列分析—ffplay.c源码分析

<div id="meta-description---">FFplay 播放器有两种队列，PacketQueue 跟 FrameQueue。FrameQueue 的数据就是从 PacketQueue 里面解码出来的（会经过滤镜）。PakceQueue 是用 FifoBuffer 来实现环形队列的，而FrameQueue 是用数组来实现一个环形队列的，但是更复杂一些。</div>

FFplay 播放器有**两种**队列，`PacketQueue` 跟 `FrameQueue`。`FrameQueue` 的数据就是从 `PacketQueue` 里面解码出来的（会经过滤镜）。

`PakceQueue` 是用 [FifoBuffer](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avfifobuffer.html) 来实现环形队列的，而`FrameQueue` 是用**数组**来实现一个环形队列的，但是更复杂一些。`FrameQueue` 跟 `PakceQueue` 的关系如下：

![1-2](https://ffmpeg.xianwaizhiyin.net/ffplay/stream_open/1-2.jpg)

------

**FrameQueue**  是一个新的数据结构，定义如下：

```
typedef struct FrameQueue {
    Frame queue[FRAME_QUEUE_SIZE]; //FRAME_QUEUE_SIZE 等于 16，队列最多缓存 16 个帧
    int rindex; //当前的读索引位置。
    int windex; //当前的写索引位置。
    int size; //这个字段不是内存大小，而是个数，代表 当前队列已经缓存了多少个 Frame。
    int max_size; //队列最多缓存多少个 Frame，max_size可能比FRAME_QUEUE_SIZE小。
    int keep_last; //播放之后是否保存上一帧在队列里面不销毁。
    int rindex_shown; //配合 keep_last 使用的。
    SDL_mutex *mutex; //SDL锁，读写队列的时候需要加锁。
    SDL_cond *cond; //SDL条件变量，用于 解码线程 跟 播放线程 通信
    PacketQueue *pktq; // 来源， FrameQueue 的数据是从哪一个 PacketQueue 里来的。
} FrameQueue;
```

------

**struct FrameQueue** 里面有几个字段是需要特别讲解一下的，如下：

**第一**，`max_size`，大伙可能会有疑问，`Frame queue[FRAME_QUEUE_SIZE]` 明明是一个固定大小的数组，这个队列的最大数量肯定是 `FRAME_QUEUE_SIZE`，为什么还要搞一个 `max_size` 出来？

**答：**`FRAME_QUEUE_SIZE` 是内存的最大值，但是队列的最大帧数是由 `max_size` 控制的，在 `frame_queue_init()`  函数初始化 `FrameQueue` 的时候，可以设置一个比 `FRAME_QUEUE_SIZE` 更小的值，如下：

![1-3](frame_queue\1-3.png)

![1-4](frame_queue\1-4.png)

------

**第二**，`rindex` 与 `windex` 字段，分别是**读索引**跟**写索引**，代表当前操作，读到数组的哪个位置，写到数组的哪个位置。

一开始的时候，`rindex` 与 `windex` 都是 0 的。而 `frame_queue_next()` 函数负责递增 `rindex` （读索引），`frame_queue_push()` 函数负责递增 `windex` （写索引）。

只有**写**进去队列了，才可以**读**，所以 `windex` 永远是跑在 `rindex` 前面的。大部分情况，解码都是很快的，会比播放速度快很多。所以两者的位置会如下：

![1-5](frame_queue\1-5.jpg)

如果 `rindex` 或者 `windex` 大于数组最大值怎么办？

**答：**`rindex` 或者 `windex` 就会回滚，变成 0 。从头开始操作。所以这其实是一个环形的队列。如下：

![1-6](frame_queue\1-6.jpg)

------

**第二**，`keep_last` 与 `rindex_shown`，这两个字段有点不容易理解。

`keep_last` 代表 播放之后是否保存上一帧在队列里面不销毁。那保存在队列里面有什么用呢？用视频流举例，当SDL窗口变小的时候，`ffplay` 可以取上一帧`Frame`，重新渲染 `texture` 来适应缩小后的窗口大小。

`keep_last` 是怎么做到保留上一帧在队列的呢？下面就让我们来探索一下。

首先，`keep_last` 不是一个可以通过命令行参数**配置**的值，可以说 `keep_last` 是在代码里面**写死**的，代表这个 `FrameQueue` 要不要保留上一帧。

而 `ffplay`，有 3 个 `FrameQueue`，分别是 视频流，音频流，字幕流。我们可以看到这 3 个流配置的 `keep_last` 是不一样的，如下：

![1-7](frame_queue\1-7.png)

只有音频 ，视频流的 `FrameQueue` 会保留上一帧，字幕流是不会保留的，这些值都是在代码里写死成 1 跟 0 。

我个人没看出来，音频流保存上一帧有何作用，有知道的读者朋友可以留意。

------

继续讲 `keep_last` 字段在代码里的逻辑。可以使用 `clion` 的 **Find Usages** 功能，可以**很精准地找到使用的地方**，如下：

![1-8](frame_queue\1-8.png)

![1-9](frame_queue\1-9.png)

可以看到，只有 `frame_queue_next()` 函数使用了这个变量，代码如下：

![1-9-2](frame_queue\1-9-2.png)

`frame_queue_next()` 函数的作用，就是把读索引`+1`，然后释放 `AVFrame` 的**引用内存**，只要 `+1` 了，后面的操作能一直往前读 `AVFrame`。

但是  `frame_queue_next()` 一开始有一个 `if` 判断，对于音频/视频流，`keep_last` 一开始是 1，而 `rindex_shown` 是 0，所以就会把 `rindex_shown`  置为 1 了，然后 `return`，**直接返回了**。

第一次是没有 把读索引 `+1`，也没有把 `AVFrame` 释放。直接返回了。

这里读者可能会疑惑，如果读索引不变，如果下次读数据，**直接用**读索引，不就拿到的还是上一帧了吗？

**答：**没错，**所以 ffplay 不是直接用读索引**，需要加上 `rindex_shown` 。

我们来看一下，`ffplay` 是怎么读取 `FrameQueue` 队列的数据的，就在 `frame_queue_peek_readable()` 函数里面，如下：

![2-1](frame_queue\2-1.png)

先不用管后面的 `% f->max_size` 取余操作，这是读索引大于 `max_szie` 就回滚的操作。因为 加了 `rindex_shown`，所以最后的值可能大于 `max_szie`。

从上图可以看出来，是通过 `rindex` 加上 `rindex_shown` 来操作的。当播放往第一帧的时候，`frame_queue_next()` 也执行完毕了。但是由于是第一帧，所以 `rindex` 就不会  `+1`， `rindex` 还是 `0` 。但是 `rindex_shown` 这个变量变成 `1` 了，所以这样取，还是能顺利读到第二帧的`Frame`。

所以，`keep_last` 是为了控制 `rindex_shown` 变成 1 的，而 `rindex_shown` 是为了实现保留上一帧在队列，还能顺利往前继续读数据的功能。

因为，我个人觉得 `FFplay` 播放器里面，其实有**两个读索引**。

第一个**读索引** 也就是 `rindex`，这其实是用来读取上一帧**已经播放**的`AVFrame`的。

第二个**读索引** 也就是 `rindex+rindex_shown`，这个是用来读取下一个**准备播放**的 `AVFrame` 的：

------

下面介绍一个几个跟 FrameQueue 相关的函数：

------

**struct FrameQueue** 数据数据至此已经讲解完毕了，下面简单介绍一下跟 `FrameQueue` 相关的函数。

**1，**`frame_queue_init()`，初始化 `FrameQueue` 的函数。

**2，**`frame_queue_peek_next()`，读取当前准备播放的帧的**下一个帧**。

**3，**`frame_queue_peek_last()`，读取上一帧**已经播放**的`Frame`

**4，**`frame_queue_peek_writable()`，peek 出一个可以写的 `Frame`，此函数可能会阻塞。

**5，**`frame_queue_peek_readable()`，peek 出一个可以准备播放的 `Frame`，此函数可能会阻塞。

**6，**`frame_queue_push()`，偏移 `windex` （写索引），+1。

**7，**`frame_queue_next()`，偏移 `rindex` （读索引），+1。

**8，**`frame_queue_last_pos()`，获取当前播放到文件的那个位置，位置是内存数据的位置。例如 100M 的mp4，播放到了 50M。

**9，**`frame_queue_destory()`，销毁`FrameQueue` 的函数。

------

注意 `FrameQueue` 队列相关函数用的动词是 `peek` ，而 `PakcetQueue` 队列相关函数用的动词是 `get`。

这两种不同的命名其实也是有意为之的。编程经验丰富的程序员基本都会知道，`peek` 代表偷看，只是看一下队列的数据，大部分情况并不会把队列的数据销毁。

在 `FFplay` 播放器里面，peek 也是代表偷看的意思，如果你一直调 `frame_queue_peek_next()` 读取到的都是同一帧，如果想读到下一帧，就需要手动调 `frame_queue_next()` 偏移一下，如下：

```
while(;;){
	Frame* f = frame_queue_peek_next();
	//frame_queue_next();
}
```

而对于 `PacketQueue` 队列来说，他是 `get` 操作，可以**一直读到下一帧**。

```
while(;;){
	MyAVPacketList*  = packet_queue_get();
}
```

------

所以 对于 `FrameQueue` 队列， `peek` + `next` 是分开操作的。而对于 `PacketQueue` 队列，`peek` + `next` 合成了一步 `get` 。

数据结构为什么如此设计呢？

是因为， `PacketQueue` 队列是给解码器用的，从队列拿一个包，必然需要立即丢给解码器。

而  `FrameQueue` 队列 是给 SDL 播放用的，从队列 `peek` 一个帧，不一定就需要播放，如果还没到播放时间，就不需要播放。具体推荐阅读《[视频播放线程分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_refresh.html)》
