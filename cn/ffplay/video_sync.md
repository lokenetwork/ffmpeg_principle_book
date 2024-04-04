# FFplay视频同步分析—ffplay.c源码分析

<div id="meta-description---">FFplay视频分析</div>

以音频时钟为主时钟，是最常用的同步方式，也是FFplay里面默认的同步方式。当以音频时钟为主时钟，视频 就会向音频同步。

视频播放线程，会缩短或者拉长当前视频帧的播放时长，或者丢弃视频帧来向音频同步。

---

**FFplay** 是用 `struct Clock` 数据结构记录音频流，视频流当前播放到哪里的。`Clock` 结构体的定义如下：

```
typedef struct Clock {
    double pts;           /* clock base */ 单位是妙。
    double pts_drift;     /* clock base minus time at which we updated the clock */
    double last_updated;
    double speed;
    int serial;           /* clock is based on a packet with this serial */
    int paused;
    int *queue_serial;    /* pointer to the current packet queue serial, used for obsolete clock detection */
} Clock;
```

每次取下一帧视频来播放的时候，就会更新视频时钟，记录那时候视频流播放到哪里了，如下：

![0-1](video_sync\0-1.png)

每次回调取音频数据来播放的时候，就会更新音频时钟，记录那时候音频流播放到哪里了，如下：

![0-2](video_sync\0-2.png)

提醒：`Clock::pts` 的时间单位是秒。

------

`struct Clock` 里面最难懂的字段就是 `pts_drift` ，`pts_drift` 字段的值是通过  `pts` 字段减去系统时间得到的，所以实际运行的时候 `pts_drift` 是一个很大的负数。如下：

![1-1](video_sync\1-1.png)

`pts_drift` 是一个很大的负数，会让你摸不着头脑，不知道是干什么的？我们来看一下有什么地方会用到 `pts_drift` 这个变量，如下：

![1-2](video_sync\1-2.png)

上图中的 `get_clock()` 函数是用来获取 当前视频流或者音频流播放到哪里的。后面的 `c->speed` 默认是 1 ，所以不用管。

`get_clock()` 里面会用 `av_gettime_relative()` 获取当前的系统时间，所以 `get_clock()` 的计算公式如下：

```
视频流当前的播放时刻 = 当前帧的 pts - 之前记录的系统时间 + 当前的系统时间
视频流当前的播放时刻 = 当前帧的 pts + 当前的系统时间    - 之前记录的系统时间
视频流当前的播放时刻 = 当前帧的 pts + 消逝的时间
```

`pts_drift` 字段的真正作用这样是为了能计算出**消逝的时间**，这样才能在每时每刻都能准确知道**当前**视频流或者音频的播放到哪里了。如果你想知道视频流播放到哪里了，调一下 `get_clock()` 函数即可。

所以 `pts_drift` 字段实际上是由两个字段组成，当前帧的 `pts` 跟 之前记录的系统时间，这两个字段的**信息量**合在一起存储就会看起来有点奇怪，是一个很大的负数。

我们所处的世界中的时间是**不断消逝的**，所以 `set_clock_at()` 需要同时记录上当时的系统时间，方便后面的  `get_clock()` 能计算出消逝了的时间。

------

现在已经明白了 Clock 时钟的概念，也知道 用 `get_clock()` 函数能获取到 视频流，音频流**当前的播放时刻**，下面就来正式进入本文的主题，视频同步的逻辑。

视频向音频同步的逻辑有两处，其中一处在 `compute_target_delay()` 函数里面，如下：

```
/* compute nominal last_duration */
last_duration = vp_duration(is, lastvp, vp);
delay = compute_target_delay(last_duration, is);
```

`last_duration` 代表当前帧本来，**本来**需要显示多长时间。当前帧是 指 窗口正在显示的帧。

`delay` 代表 当前帧实际，**实际**应该显示多长时间。

举个例子，1/24 帧的视频流，每帧固定显示 0.04s，当音频跟视频播放不同步的差异不超过阈值的时候，`last_duration` 跟 `delay` 都会是 0.04。

但是当视频比音频快了 0.05s 的时候，那 `delay` 就会从 0.04 变成 0.08，翻倍了，拉长当前视频帧的播放时间来等待音频流追上来。

当视频比音频慢了 0.05s 的时候，那 `delay` 就会从 0.04 变成 0，这样当前视频帧就会立即结束播放，让下一帧显示出来。

------

下面来分析一下 `compute_target_delay()` 函数的实现，如下：

![1-3](video_sync\1-3.png)

可以看到，如果是音频时钟为主时钟，就会跑进去 `if` 里面的逻辑。

变量 `diff` 代表视频时钟与主时钟的时间差，主时钟默认是音频时钟。当 diff 大于 0 的时候，代表 视频时钟 比 音频时钟 快。当 diff 小于 0 的时候，代表 视频时钟 比 音频时钟 慢。`diff` 的单位是秒。

之前讲过，音视频不同步是常态，不需要做到完全同步，只要把不同步的程度控制在阈值范围内，人就感受不到不同步了。

FFplay 里面计算同步阈值（sync_threshold）的方式有点复杂，如下：

```
sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
```

先介绍一下两个宏：

1，AV_SYNC_THRESHOLD_MIN，最小的同步阈值，值为 0.04，单位是 秒

2，AV_SYNC_THRESHOLD_MAX，最大的同步阈值，值为 0.1，单位是 秒

上面的代码，就是从 0.04 ~ 0.1 之间选出一个值作为 同步阈值。

对于 1/12帧的视频，delay 是 0.082，所以 sync_threshold 等于 0.082，等于一帧的播放时长。

对于 1/24 帧的视频，delay 是 0.041，所以 sync_threshold 等于 0.041，等于一帧的播放时长。

对于 1/48 帧的视频，delay 是 0.0205，所以 sync_threshold 等于 0.04，约等于两帧的播放时长。

这就是 FFplay 计算同步阈值（sync_threshold） 的算法。

------

计算出同步阈值之后，就需要判断 音视频的时间差 `diff` 是否超过阈值，所以就有了下面的判断：

```
if (!isnan(diff) && fabs(diff) < is->max_frame_duration) {
    if (diff <= -sync_threshold)
    	delay = FFMAX(0, delay + diff);
    else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD)
    	delay = delay + diff;
    else if (diff >= sync_threshold)
    	delay = 2 * delay;
}
```

`is->max_frame_duration` 通常是 10s，这个判断是当音视频不同步的差异超过 10s，就不再进行同步操作，不管摆烂。

`diff` 是负数的时候，代表视频比音频慢了，通常会将 `delay` 置为 0，说实话，我想不到 `FFMAX(0, delay + diff)` 能返回大于 0 的场景。

`diff` 是正数的时候，代表视频比音频快了，当超过阈值的时候，就会把 `delay * 2`。

这就是 FFplay 里面视频向音频同步的算法逻辑，不过写这段代码的作者也留了一句注释，如下：

```
We take into account the delay to compute the threshold. I still don't know if it is the best guess 
```

他也不太清楚，这样计算出来的同步阈值（sync_threshold）是否是一个最好的实现。

不过 `ffplay` 通常播放视频没有问题，证明这个算法还是可以的，感兴趣的读者可以看一些 VLC 播放器的同步实现。

---

上面讲的是视频播放的时候的同步逻辑，而在视频帧刚解码出来的时候，FFplay 也需要会进行音视频同步，不过这里不会用到同步阈值，只要视频已经比音频慢了，无论慢多少，都会立即丢帧。如下：

![1-4](video_sync\1-4.png)

`framedrop` 默认值是 **-1**。 注意 C99 标准是没有布尔值的，只要是非 0 的值，都是真，所以默认情况下，下面的条件就为真。

```
if (framedrop>0 || (framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) 默认等于 true
```

从上图可以看到，还是计算出 视频与音频的时间差  `diff` 。

下面这个判断比较复杂，有好几个条件符合才会进行丢帧。

```
if (frame->pts != AV_NOPTS_VALUE) {
    double diff = dpts - get_master_clock(is);
    if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
        diff - is->frame_last_filter_delay < 0 &&
        is->viddec.pkt_serial == is->vidclk.serial &&
        is->videoq.nb_packets) {
            is->frame_drops_early++;
            av_frame_unref(frame);
            got_picture = 0;
    }
}
```

1，音视频时间差 `diff` 不超过10s（AV_NOSYNC_THRESHOLD ）

2，序列号一致，就是为了快进快退功能服务的。

3，`PacketQueue` 队列里面还有数据可以解码。

4，视频比音频播放慢了，也就是 `diff - is->frame_last_filter_delay < 0` 条件。

第四个条件是最难懂，为什么要减去 `frame_last_filter_delay ` 呢？

`diff` 只要小于 0 了，就代表视频比音频播放慢了，这个无可厚非，`diff` 小于 0 ，它减去 `frame_last_filter_delay` ，必然还是小于 0 。

所以减去 `frame_last_filter_delay `是为了服务一些 `diff` 大于 0 的情况。

首先 `frame_last_filter_delay` 变量存储的是滤镜容器处理上一帧所花的时间，这是一个预估值，假设滤镜容器处理上一帧花了 0.01s，那处理现在这一帧估计也需要0.01s，所以解码出来的 `AVFrame`，并不是立即就能丢进去 `FrameQueue` 给播放线程用。而是需要经过滤镜处理的，滤镜处理也需要时间。

所以如果 `diff` 等于 0.008 ，视频比音频快了 0.008s，但是因为视频要经过滤镜处理，所以需要减去 0.01 ，实际上是 视频比音频播放慢了 0.002s。

```
0.008 -0.01 = -0.002
```

最后会用 `is->frame_drops_early` 来记录一下这种情况的丢帧数量。

------

最后还有一个视频同步的地方，就是 `is->frame_drops_late`，但是这里没有用到音频时钟。

![1-5](video_sync\1-5.png)

这段代码的逻辑是在 `video_refresh()` 视频播放线程里面的，当从 `FrameQueue` 队列拿到一个帧的时候，会判断这个帧是否已经过了它的播放时间，如果过了就丢帧。

最后会用 `is->frame_drops_late`  来记录一下这种情况的丢帧数量。

------

总结，视频同步一共有三处地方。

1，解码出来之后。

2，播放的时候，检测视频帧是否已经过了它的播放时间。

3，播放的时候，`compute_target_delay()` 让当前帧立即结束播放，或者延迟播放时间。
