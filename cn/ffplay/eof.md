# FFplay播放完毕分析—ffplay.c源码分析

<div id="meta-description---">当FFplay播放完毕的时候，各个解复用线程，解码线程，播放线程在干什么。</div>

FFplay 播放器播放完一个 mp4 文件的时候，画面就会停止在最后一帧。如下：

![1-1](eof\1-1.png)

本文主要介绍，当播放完毕的时候，各个解复用线程，解码线程，播放线程在干什么。

------

[main主线程/视频播放线程](https://ffmpeg.xianwaizhiyin.net/ffplay/video_refresh.html)，播放完毕之后，它在干什么？

播放完一个文件之后，`main` 主线程实际上是没有受到什么影响的，还是不断在 `event_loop()` 里面循环，检测键盘/窗口事件有没发生，大部分情况每隔 0.01s 调一次 `video_refresh()` 看看有没下一帧数据可以播放。

但是由于已经播放完毕，所以 `FrameQueue` 里面是空的，所以 `video_refresh()` 基本上相当于什么都没做。

![1-2](eof\1-2.png)

虽然已经播放完毕，但是还是可以按 🠔 键 后退到某个时间点，后退之后， `FrameQueue` 就会又有数据，这样  `video_refresh()`  就会继续播放画面。

---

[read_thread解复用线程](https://ffmpeg.xianwaizhiyin.net/ffplay/read_thread.html)，播放完毕之后，它在干什么？

![1-3](eof\1-3.png)

当 `read_thread` 解复用线程 读到 `AVERROR_EOF` 的时候，就会往解码器丢 **空的** `AVPacket` ，这样可以让解码器把所有缓存的帧都刷出来去播放。

`is->eof` 变量代表 mp4 文件是否已经播放完毕。

最后会调 `SDL_CondWaitTimeout()` 休眠 10ms，然后再继续跑，不断调 `av_read_frame()` 函数，看看能不能读到数据。

当后退，跳转时间点播放的时候，`av_read_frame()` 就能重新读到数据了。

------

[video_thread视频解码线程](https://ffmpeg.xianwaizhiyin.net/ffplay/video_thread.html)，播放完毕之后，它在干什么？

由于 read_thread解复用线程 一直读不出来 `AVPacket` ，所以 `PacketQueue` 会一直没有数据能拿出来解码，`PacketQueue` 队列是空的。

`PacketQueue` 队列没有数据，就会导致 `video_thread` 线程一直**阻塞**在 `packet_queue_get()` 函数里面。函数调用流程如下：

![1-4](eof\1-4.jpg)

----

[audio_thread音频解码线程](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_thread.html)，播放完毕之后，它在干什么？

audio_thread 跟 video_thread 一样，音频流的 `PacketQueue` 队列也没有数据，所以也会一直**阻塞**在 `packet_queue_get()` 函数里面。

![1-5](eof\1-5.jpg)

---

[sdl_audio_callback音频播放线程分析](https://ffmpeg.xianwaizhiyin.net/ffplay/sdl_audio_callback.html)，播放完毕之后，它在干什么？

因为文件播放完毕 ，导致 `FrameQueue` 队列空了，所以 `sdl_audio_callback` 会阻塞在 `frame_queue_peek_readable()` 函数里面，如下：

![1-6](eof\1-6.png)

![1-7](eof\1-7.jpg)

------

总结，当播放完一个 mp4 的时候，各个线程状态如下：

1，`main`主线程，没什么影响，还是正常地处理事件。

2，`read_thread` 解复用线程，会进行**短暂的休眠**（10ms）

3，`video_thread` 视频解码线程，会进行**永久休眠**，阻塞在 `packet_queue_get()` 函数里面。

4，`audio_thread` 音频解码线程，会进行**永久休眠**，阻塞在 `packet_queue_get()` 函数里面。

5，`sdl_audio_callback` 音频播放线程，会进行**永久休眠**，阻塞在 `frame_queue_peek_readable()` 函数里面。

上面这些永久休眠，都是可以通过 `cond` 条件变量来唤醒的。

分析完毕。
