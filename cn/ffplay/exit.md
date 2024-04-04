# FFplay退出分析—ffplay.c源码分析

<div id="meta-description---">FFplay退出逻辑分析</div>

当 FFplay播放器播放完一个 mp4 文件的时候，画面就会停止在最后一帧，并不会自动退出，如下：

![1-1](https://ffmpeg.xianwaizhiyin.net/ffplay/eof/1-1.png)

有两种方式可以退出 FFplay 播放器。

**1，**在命令行使用 `autoexit` 参数，这是一个布尔参数，是没有值的，使用如下：

```
ffplay -autoexit -i juren.mp4
```

当指定了 `autoexit` 的时候，播放到结尾，FFplay 会自动退出。 如果不想自动退出，可以前面加上 no 前缀，如下：

```
ffplay -noautoexit -i juren.mp4
```

其实使不使用 `noautoexit` 参数都是一样的，因为这是默认值，默认不会自动退出。

**这是 FFmpeg 整个项目布尔参数的习惯，如果要取反，就在前面加个 `no`**。

`ffmpeg.exe`，`ffplay.exe` ，`ffprobe.exe`  这些可执行文件的命令行布尔参数都是这样的。

------

还有另一种方法可以退出  `FFplay` 播放器，那就是按 `Q` 键，

先来看一下处理 `Q` 键 的代码，如下：

![1-1](exit\1-1.png)

`do_exit()` 函数的实现如下：

![1-2](exit\1-2.png)

`do_exit()` 函数实际上只有 **两部分**，调 `stream_close()` 跟 释放数据。后面的都是释放各种数据结构。流程图如下：

![1-3](exit\1-3.jpg)

`do_exit()` 里面有两个变量的使用是特别重要的，会连环影响几个地方的逻辑。

1，`is->abort_request`

2，`q->abort_request`

`q->abort_request` 跟 `is->abort_request` 其实是一样的，都是会在退出播放器的时候置为1，只是为了方便调用，所以在 PacketQueue 里面也加了同样的字段 `abort_request` 。

------

先来看一下用到 `is->abort_request` 的地方，如下：

![1-4](exit\1-4.png)

可以看到，当  `is->abort_request` 为 1 的时候，就会跳出 `for` 死循环，然后 read_thread 线程自然就结束了。

还有一个地方也会用到  `is->abort_request` ，那就是 `decode_interrupt_cb()` ，这个是封装层的回调函数。

```
static int decode_interrupt_cb(void *ctx)
{
    VideoState *is = ctx;
    return is->abort_request;
}
```

在播放网络流的时候，网络卡顿，`av_read_frame()` 可能要 8 秒才能返回，这时候如果想关闭播放器，就需要 `av_read_frame()` 尽快地返回，不要再阻塞了。这时候，就需要 `interrupt_callback` 了，因为在 8 秒 内，`av_read_frame()` 内部也会定时执行 `interrupt_callback()`，只要 `interrupt_callback()` 返回 1，`av_read_frame()` 就会不再阻塞，立即返回。

提醒：播放网络流的时候，`avformat_find_stream_info()` 可能会跟 `av_read_frame()` 一样阻塞很久。

------

接下来看一下用到 `q->abort_request` 的地方，如下：

![1-5](exit\1-5.png)

可以看到有 11 个地方用到这个  `q->abort_request` 。

这 11 个地方比较繁琐，主要的作用就是让解码线程跳出死循环循环，然后退出。

------

`stream_component_close()` 里面的 `decoder_abort()` 函数也比较重要，就是把 `q->abort_request`  设置为 1，然后通过 `cond` 唤醒各个线程，各个线程发现  `q->abort_request`  为 1，就会自觉退出。

```
static void decoder_abort(Decoder *d, FrameQueue *fq)
{
    packet_queue_abort(d->queue);
    frame_queue_signal(fq);
    SDL_WaitThread(d->decoder_tid, NULL);
    d->decoder_tid = NULL;
    packet_queue_flush(d->queue);
}
```

------

FFplay 播放器退出处理分析完毕。
