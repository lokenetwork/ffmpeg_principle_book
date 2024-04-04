# FFplay暂停分析—ffplay.c源码分析

<div id="meta-description---">暂停也是播放器非常常见的功能。对于 FFplay 播放器，可以通过 p 键 或者空格键 来切换暂停状态。</div>

暂停也是播放器非常常见的功能。对于 FFplay 播放器，可以通过 `p` 键 或者空格键 来切换暂停状态。

先来看一下处理 `p` 键 的代码，如下：

![1-1](pause\1-1.png)

从上图可以看到，调了 `toggle_pause()` 函数，注意这个 `cur_stream` 参数，其实这个参数不是视频流或者音频流，而是 FFplay 的全局管理器 `VideoState`

`toggle_pause()` 函数的实现如下：

```
static void toggle_pause(VideoState *is)
{
    stream_toggle_pause(is);
    is->step = 0;
}
```

后面为什么会把 `is->step` 置为 0 ？`is->step` 变量是给逐帧播放用的，为 0 代表退出逐帧播放模式。

这是因为逐帧播放实际上就是用切换暂停状态来实现的，每播放一帧就立即暂停。具体请看《[FFplay逐帧播放分析](https://ffmpeg.xianwaizhiyin.net/ffplay/step.html)》

------

再来看一下 `stream_toggle_pause()` 函数的实现，如下：

![1-2](pause\1-2.png)

从 启动状态 切换到 暂停状态的时候，`is->paused` 等于 0，所以是不会跑进去 `if (is->paused) {...}` 里面的。

上图中用 `set_clock()` 更新了**外部时钟**，`set_clock()` 函数会把 `Clock::pts` 更新到当前最新的播放时刻。

通常情况下，`get_clock()` 函数获取当前的最新播放时刻，是用 `Clock::pts`  + 消逝的时间。

但是消逝的时间可以是 0 的，什么情况下消逝的时间是 0 呢？就是暂停的时候，当播放器暂停的时候，他外部时钟也会暂停，所以消逝的时间为 0 ，如下：

![1-3](pause\1-3.png)

`stream_toggle_pause()` 函数才需要更新外部时钟的 pts 成最新的值，如果不更新，在暂停状态下获取到的外部时钟播放时刻就不准。

在暂停状态下，哪段代码还会调 `get_clock()` 获取外部时钟的播放时刻呢？这个后面揭晓。

 `stream_toggle_pause()` 函数最后还会把 4 个暂停变量都被设置成 1，或者 从 1 切换成 0 。如下：

```
is->paused = is->audclk.paused = is->vidclk.paused = is->extclk.paused = !is->paused;
```

我们来看一下这  4 个暂停变量会影响哪些代码逻辑？

首先是 **main** 主线程，主线程主要是处理键盘事件 跟 播放视频画面，键盘事件不会受到暂停状态影响，该处理还是继续处理。

播放视频画面 的函数是 `video_refresh()` ，但是暂停状态下也不会调  `video_refresh()` ，如下：

![1-4](pause\1-4.png)

但是暂停状态下，如果改变了 `ffplay` 窗口大小，`is->forece_refresh` 机会变成 1，就会调   `video_refresh()` 。

因此 **main** 主线程，在暂停状态下，只会不断检测，处理键盘事件以及一些窗口事件，大部分时候并不会调 `video_refresh()` 播放视频画面。

------

再来看一下 [read_thread解复用线程](https://ffmpeg.xianwaizhiyin.net/ffplay/read_thread.html)，在暂停状态下，它在干什么？

![1-5](pause\1-5.png)

上图的是为了兼容一些网络流的播放，有些流媒体协议支持暂停跟播放操作，当暂停的时候，服务器端就不会再推流过来了。对于本地播放，上面的代码是没用的。

我翻了一圈 `read_thread()` 函数的代码，发现它并不会因为 pause 变成 1 而停下，`read_thread()` 线程即使在暂停状态下，也是不断运行，不断读取数据，直到 塞满队列，塞满队列就会休眠 10 ms，如下：

![1-6](pause\1-6.png)

感兴趣的读者可以在暂停状态下往 `SDL_CondWaitTimeout()` 那里打个断点，会不断跑进去那里的逻辑。

------

再来看一下 [video_thread视频解码线程](https://ffmpeg.xianwaizhiyin.net/ffplay/video_thread.html)，在暂停状态下，它在干什么？

研究 `video_thread()` 函数的代码，我们发现了第一个在暂停状态下，获取外部时钟的地方，也就是 `video_thread()` 里面的 `get_video_frame()` 函数

![1-7](pause\1-7.png)

当外部时钟是 主时钟的时候，这里的 `get_master_clock()` 获取的就是外部时钟的播放时刻。因此如果在 `stream_toggle_pause()` 里面不更新外部时钟，这里获取到的时间就是错误的，会导致误判，丢帧。

翻了一圈  `video_thread()` 解码线程的代码，发现也不会受到 paused 的影响，还是会正常解码，但是如果塞满 `FrameQueue` 队列的时候，就会一直阻塞在 `frame_queue_peek_writable()` 函数里面。

---

再来看一下 [audio_thread音频解码线程](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_thread.html)，在暂停状态下，它在干什么？

研究发现，跟 `video_thread()` 视频解码类似，也不会受到 paused 的影响，还是会正常解码，但是如果塞满 `FrameQueue` 队列的时候，就会一直阻塞在 `frame_queue_peek_writable()` 函数里面。

------

最后来看一下 [sdl_audio_callback音频播放线程分析](https://ffmpeg.xianwaizhiyin.net/ffplay/sdl_audio_callback.html)，在暂停状态下，它在干什么？

首先，`sdl_audio_callback()` 是回调函数，所以你可以猜到，肯定不会阻塞，肯定会返回去给 SDL 。

首先，`sdl_audio_callback()` 是会受到 paused 的影响的，如下：

![1-8](pause\1-8.png)

![1-9](pause\1-9.png)

暂停状态下，`audio_decode_frame()` 函数会直接返回 -1，就会导致 输出静音数据，只要直接把 stream 指向的内存数据设置为 0 就是输出静音数据了。

```
memset(stream, 0, len1);
```

虽然是输出静音数据，但是音频播放线程还是在跑的，没有阻塞，她在跑，就会更新音频时钟，如下：

![2-1](pause\2-1.png)

虽然会更新音频时钟，但是因为在暂停状态下没有跑进去 `audio_decode_frame()`，所以 `is->audio_clock` 没有更新。因此即便更新音频时钟，也是用原来的值来更新的。

所以音频时钟相当于没有更新。

读者可以在 `sdl_audio_callback()` 入口加个日志，如下：

![2-2](pause\2-2.png)

会发现虽然音频时钟一直在跑，但是这个 `get_clock()` 返回的值是没有增长的。

![2-3](pause\2-3.png)

------

总结一下，从启动状态切换到暂停状态，影响的地方。

1，导致视频播放函数没有调用，导致 `FrameQueue` 堆积，所以视频解码线程会阻塞在 `frame_queue_peek_writable()` 函数里面

2，导致音频播放线程直接输出静音数据，没有从 `FrameQueue` 读数据，导致 `FrameQueue` 堆积，所以音频解码线程会阻塞在 `frame_queue_peek_writable()` 函数里面。

3，音频解码线程，视频解码线程阻塞，一直不从 `PacketQueue` 拿数据去解码，导致 `PacketQueue ` 堆积，从而导致 `read_thread`解复用线程不再从文件读取数据了。

------

回到 `stream_toggle_pause()` 函数，当从暂停状态切换到启动状态的时候，有一段逻辑非常奇怪，如下：

![2-4](pause\2-4.png)

可以看到，它会更新 `is->frame_timer` 跟 视频时钟，为什么要这么做呢？

因为 `is->frame_timer` 的单位是系统时间，代表窗口现在这一帧画面是在什么系统时间开始播放的。更新 `is->frame_timer` 之后代表窗口当前画面是从此刻开始播放的。

暂停之后，系统时间是一直在跑的。例如，暂停 8 秒钟之后，再调 `av_gettime_relative()` 会发现比之前多了 8 秒。

因此切换回来的时候，要及时更新  `is->frame_timer` ，这个变量的含义不能变的。如果你不更新，就代表视频播放线程卡顿，没有调度过来，导致的这帧画面播放了 8 秒，虽然他确实是播放了 8秒，但是不能体现在 `is->frame_timer` 变量上面。

后面在 `video_refresh()` 里面会用  `is->frame_timer`  来判断是否已经超过**预定的播放时刻**，超过了就会丢帧，如下：

![2-5](pause\2-5.png)

上图中，只有超过 10 秒才会纠正  `is->frame_timer` ，如果你只暂停了 8 秒就不会纠正。

因此，如果在 暂停状态切换到启动状态的时候，不更新  `is->frame_timer` ，就会导致 8 秒的视频帧被丢弃，因为会误判它们都已经过了预定的播放时刻。

------

再来看一下，从暂停状态切换到启动状态的时候，为啥需要更新视频时钟，如下：

![2-6](pause\2-6.png)

注意，他是先把视频时钟的暂停状态恢复，然后再更新时钟的，所以 `get_clock()` 获取的是 pts + 消逝的时间，消逝的时间其实就等于暂停了多久。

其实我也不太清楚 这里更新 视频时钟的 pts 的具体作用，因为暂停状态已经恢复了，随意即使这里不执行 `set_clock()`，`get_clock()` 获取到的也是正确的播放时刻。

个人猜测是字幕的场景用的，因为那里用了 clock 里面的 pts 字段。

![2-7](pause\2-7.png)

------

至此，`ffplay` 播放器暂停功能分析完毕。
