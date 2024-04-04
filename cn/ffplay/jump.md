# FFplay跳转时间点播放—ffplay.c源码分析

<div id="meta-description---">播放器的最常用的功能之一 就是快进快退，快进快退的本质就是让视频文件跳转到另一个时间点来播放。</div>

播放器的最常用的功能之一 就是快进快退，快进快退的本质就是让 mp4文件 跳转到另一个时间点来播放。

FFplay 播放器有两种方式可以让 mp4文件 跳转到另一个时间点来播放。

**1，**在命令行里里使用 `-ss` 参数，如下，跳到到第 60 秒的地方开始播放。

```
ffplay -x 400 -ss 60 -i juren.mp4
```

**2，**在播放过程中，按上下左右减，进行快进后退操作。

上面这两种跳转时间点播放的方式，其实都是调 [avformat_seek_file()](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avformat_seek_file.html) 来实现的，都是在 [read_thread()](https://ffmpeg.xianwaizhiyin.net/ffplay/read_thread.html) 线程函数里面进行 seek 操作的。

------

先来讲解命令行的  `-ss` 的原理，代码如下：

![1-1](jump\1-1.png)

可以看到，是用 `opt_seek()` 函数来接受 `-ss` 参数的，代码如下：

提示：更多命令行参数解析的细节，请阅读《[parse_options命令行参数解析](https://ffmpeg.xianwaizhiyin.net/ffplay/parse_options.html)》

![1-2](jump\1-2.png)

可以看到，最后赋值到了 全局变量 `start_time`，然后 `start_time` 变量会在 `read_thread()` 函数入口不远的地方被使用，如下：

![1-3](jump\1-3.png)

注意，上面的 seek 是在  `for(;;) {...}` 循环之前的。

------

接着讲一下播放过程中 快进后退的实现。当按下 ↓ 键的时候，就会后退进 60s ，看一下代码实现：

![1-5](jump\1-5.png)

从上图可以发现，有两种 seek 方式，一种是按时间 seek （默认），一种是 按字节 seek。

如果是按字节 seek，就会把时间转成字节位置。

无论是字节 seek，还是时间 seek，都是绝对的。例如绝对时间 跟 绝对位置。

`frame_queue_last_pos()` 获取当前视频流，或者音频流播放到哪个字节位置了，再加上 incr （相对）。

`get_master_clock()` 获取当前主时钟播放到第几秒了，然后再加上 incr （相对）。

最后调 `stream_seek()` 记录要跳转的位置，注意 `stream_seek()` 只是记录一下位置，并不会进行跳转操作。

![1-6](jump\1-6.png)

那 `is->seek_req` 跟  `is->seek_pos` 会在哪里被使用呢？在 `read_thread()` 里面。

在播放一个 mp4 文件的时候，`read_thread()` 会一直阻塞在 `for(;;) {...}` 循环里面，不断调 `av_read_frame()` 读取 `AVPacket` 出来。如下：

![1-4](jump\1-4.png)

但是在 调 `av_read_frame()` 之前，会检测一下  `is->seek_req` 是否为 1，为 1 代表有 seek 请求发生，如下：

![1-7](jump\1-7.png)

上图中，`avformat_seek_file()` 之后，就需要用 `packet_queue_flush()` 清空之前队列里面的缓存，还有刷新序列号。序列号的内容请看《[FFplay序列号分析](https://ffmpeg.xianwaizhiyin.net/ffplay/serial.html)》

`seek` 之后还会更新外部时钟，因为外部时钟就是预定的播放时刻，跳转的就是预定的播放时刻。

```
if (is->seek_flags & AVSEEK_FLAG_BYTE) {
	set_clock(&is->extclk, NAN, 0);
} else {
	set_clock(&is->extclk, seek_target / (double)AV_TIME_BASE, 0);
}
```

------

FFplay跳转时间点播放 分析完毕。
