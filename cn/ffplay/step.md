# FFplay逐帧播放分析—ffplay.c源码分析

<div id="meta-description---">FFplay 播放器有一个比较有趣的功能，就是逐帧播放。因为平时视频文件的帧率是挺高的，一秒24帧，有些细节一瞬间就过去了，不太容易注意到。利用逐帧播放功能，你可以一帧一帧的观察视频画面，这在一些查处交通违规，案件排查的场景非常有用。</div>

FFplay 播放器有一个比较有趣的功能，就是逐帧播放。因为平时视频文件的帧率是挺高的，一秒24帧，有些细节一瞬间就过去了，不太容易注意到。

利用逐帧播放功能，你可以一帧一帧的观察视频画面，在查处交通违规，案件排查的场景非常有用。

你可以在 `FFplay` 播放器运行过程中，按 `S` 键进入**逐帧播放模式**，不断按 `S` 键可以逐步看下一帧的内容，如果想重新开始播放视频，直接按 `P` 键即可，之前说过，**逐帧播放其实是用暂停功能实现的**。

------

下面就来分析一下 逐帧播放的实现原理。

先来看一下 处理 `S` 键事件的代码，如下：

![1-1](step\1-1.png)

再来看一下 `step_to_next_frame()` 函数的实现：

```
static void step_to_next_frame(VideoState *is)
{
    /* if the stream is paused unpause it, then step */
    if (is->paused)
        stream_toggle_pause(is);
    is->step = 1;
}
```

可以看到，如果当前已经是暂停状态，这个函数就会用 `stream_toggle_pause()` 把它恢复成启动状态。

最后设置 `is->step` 为 1，代表进入了逐帧播放模式。

下面来看一下逐帧播放模式下的逻辑是怎样的。

再次强烈推荐一下 `clion` 的 `Find Usages` 功能，可以快速找到使用 `is->step` 的地方。

![1-2](step\1-2.png)

可以看到，逐帧模式下，是不会进行 **late 丢帧**的，如下：

![1-3](step\1-3.png)

这样也是合理的，因为在排查案件的时候，肯定每一帧数据都要看。

------

进入逐帧模式，会先设置 `is->paused` 为 0。

当 `is->step` 为 1， `is->paused` 为 0 的时候，就会触发 `video_refresh()` 函数里面的操作，如下：

![1-4](step\1-4.png)

可以看到在调 `stream_toggle_pause(is)` 之前，已经把 `Frame` 从 `FrameQueue` 提取出来了，而 `stream_toggle_pause()` 会把  `is->paused` 为 1，也就是播放完一帧之后就立即进入暂停状态。

------

总结，逐帧播放的逻辑如下：

1，按 `S` 进入逐帧播放模式，（设置 `is->step` 为 1）。

2， `step_to_next_frame()` 把播放器状态从暂停恢复成启动。这样才能从 `refresh_loop_wait_event()` 跑进去 `video_refresh()` 函数。

3， `video_refresh()` 函数播放完一帧之后，判断是否当前是否是逐帧模式（`is->step` 是否为 1），如果是逐帧播放模式，播放完下一帧之后，立即进入暂停状态。

4，按 `P` 键会调用 `toggle_pause()` ，把  `is->step` 设置为 0，即退出逐帧播放模式。
