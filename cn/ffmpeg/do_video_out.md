# do_video_out视频编码封装—ffmpeg.c源码分析

<div id="meta-description---">如果不关注那 3 块复杂逻辑，do_video_out() 是比较简单的，就是从 buffer sink 读取 `AVFrame`，然后进行编码，转换时间基，写入文件就完事。</div>

学习一个函数，第一步是看他里面的局部变量，理解了局部变量的用途，基本也就了解了函数的功能。

`do_video_out()` 函数的局部变量如下，（只选取一部分重点的局部变量做讲解）。

```
int ret, format_video_sync;
AVPacket *pkt = ost->pkt;
AVCodecContext *enc = ost->enc_ctx;
AVRational frame_rate;
int nb_frames, nb0_frames, i;
double delta, delta0;
double duration = 0;
double sync_ipts = AV_NOPTS_VALUE;
int frame_size = 0;
InputStream *ist = NULL;
AVFilterContext *filter = ost->filter->filter;
```

**1，**`format_video_sync`，视频同步方式，我不太明白为什么要叫 sync （同步），实际我个人觉得，这是视频的**存储方式**。

这个变量实际上是一个枚举，有 5 个值，如下：

```
#define VSYNC_AUTO       -1
#define VSYNC_PASSTHROUGH 0
#define VSYNC_CFR         1
#define VSYNC_VFR         2
#define VSYNC_VSCFR       0xfe
#define VSYNC_DROP        0xff
```

**`VSYNC_PASSTHROUGH`，**代表不改变视频的**存储方式**，也就是不会改变视频帧的 pts，输入是怎样的时间，输出还是怎样的时间。

**`VSYNC_DROP`，**跟 `VSYNC_PASSTHROUGH` 非常类似，但是会在 `write_packet()` 里把 `pts` ，`dts` 破坏掉，这样 `av_interleaved_write_frame()` 内部会按照帧率来设置 `pts`，如下：

![1-1](do_video_out\1-1.png)

**`VSYNC_CFR`，**代表强制输出流为**恒定帧率模式**，CFR 的全称是 constant frame rate （恒定帧率）。恒定帧率就是说，如果是 1/24 的帧率，那一秒钟肯定有 24 帧数据保存在文件里面。

**`VSYNC_VFR`，**与恒定帧率模式对应的就是**可变帧率模式**，VFR 的全称是 variable frame rate （可变帧率）。可变帧率可以让文件体积变小。

例如：本来这个视频的帧率是 1/24，但是在某一个时间段这个画面都不会动，就可以降低采样，每秒只捕捉10帧，类似这样，因此虽然文件封装层记录的帧率是 1/24，但是不一定每一秒都有 24 帧数据。1/24 只是代表每一秒**最多**有 24 帧。

**`VSYNC_VSCFR`**，ffmpeg.exe 的这个模式我具体也不知道是干什么的，但是这个状态是从 CFR 转换过来的，有两种情况会 从 CFR 换成 VSCFR，如下：

- 文件只有一个输入流，而且命令没有使用 `-itsoffset` 选项。
- 命令行使用了 `-copyts` 选项，

![1-3](do_video_out\1-1-2.png)

从代码看，`VSYNC_VSCFR` 的规则好像只对第一帧生效。

![1-2](do_video_out\1-2.png)

---

**2，**`AVPacket *pkt`，用来接受编码器吐出来的 `AVPacket` 

**3，**`AVRational frame_rate`，从 buffer sink 出口滤镜里读取到的帧率。

**4，**`int nb_frames`，要输入给编码器的帧数量，初始值是 1，但可能会变成大于 1 的数，代表重复输入一帧给编码器，内容是一样的，但是 `pts` 不一样。

通常 VFR 转成 CFR 的时候 `nb_frames` 会大于 1。或者使用 `-r` 选项变大帧率也会导致 `nb_frames` 大于 1。

`nb_frames` 也有可能变成 0，变成 0 会导致丢弃该帧，不输入给编码器，例如使用  `-r` 选项降低帧率的时候就会出现这种情况。

**5，**`int nb0_frames`，为 CFR 功能服务的，只有输出流为 CFR 模式，这个 `nb0_frames` 才不为 0。

![1-3](do_video_out\1-3.png)

**5，**`double sync_ipts`，与从滤镜里面读取出来的 `AVFrame` 的 pts 是一样的，只是单位不一样，不过注意 `sync_ipts` 是 double 类型，而且在 `adjust_frame_pts_to_encoder_tb()` 函数做过处理，精度比较高。

**6，**`double delta0`，这个字段是 用 `sync_ipts` 减去 `ost->sync_opts` 计算出来的。

**7，**`double delta`，这个字段的其中一个作用是降低帧率的时候实现丢帧，在 VFR 模式下，如果 delta 小于 `-0.6` ，就会把 `nb_frames` 变成 0 ，实现丢帧，如下：

![1-4](do_video_out\1-4.png)

**局部变量讲解完毕。**

----

`do_video_out()` 这个函数里面有 3 块逻辑 非常复杂。

**1，**sync 同步模式的处理，CFR 与 VFR 的相互转换，等等。推荐阅读《[FFmpeg可变帧率转恒定帧率详解](https://ffmpeg.xianwaizhiyin.net/ffmpeg/vfr_to_cfr.html)》

**2，**强制关键帧逻辑，命令行 `-force_key_frames` 选项的功能，下图这一大块逻辑都是为这个功能服务的：

![1-5](do_video_out\1-5.png)

强制关键帧逻辑的详细介绍推荐阅读《[FFmpeg强制关键帧分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/force_keyframe.html)》

**3，**帧率变换逻辑，命令行 `-r` 选项的功能，可以降低帧率与提高帧率。推荐阅读《[FFmpeg是如何调整帧率的](https://ffmpeg.xianwaizhiyin.net/ffmpeg/change_framerate.html)》

**sync 同步模式逻辑** 与 **帧率变换逻辑** 其实是共用一些变量的，例如 `nb0_frames`，`nb_frames`，`delta0`，`delta`。

这些变量特别复杂，不结合实际的命令来调试分析，根本看不懂。

---

不过本文建议读者不需要关注上面这 3 块逻辑，因为本文讲解的是简单场景，使用的命令如下：

```
ffmpeg -i juren-5s.mp4 juren-5s-2.mp4
```

这条命令不涉及到 CFR 转 VFR，也没有使用  `-force_key_frames` 强制关键帧，也没用 `-r` 降低帧率。

所以上面这 3 块逻辑的代码，在简单场景下，是没用的。初学者不需要看。

---

除了 3 大复杂逻辑，`do_video_out()` 函数里面还有哪些重点呢？

**第一个重点**：限制输出流的帧数量，可以用命令行选项 `-frames` 指定最大帧数 `max_frames`，代码如下：

```
nb_frames = FFMIN(nb_frames, ost->max_frames - ost->frame_number);
```

可以看到当超过最大帧数之后，`nb_frames` 会变成负数，所以就会丢弃帧。

---

在简单场景，`do_video_out()` 只会输入一个 `AVFrame` 给编码器进行编码，所以只看下面这个 for 循环就可以了。

```
/* duplicates frame if needed */
for (i = 0; i < nb_frames; i++) {
	...省略代码...
}
```

从 `for` 循环里面可以看到，`pts` 会被替换，如下：

```
in_picture->pts = ost->sync_opts;
```

但是在简单命令下，**这跟没替换是一样的**，如下：

![1-6](do_video_out\1-6.png)

`ost->sync_opts` 本来就是 从 `sync_ipts` 里来的。

**在本文的简单命令中，这个 delta 变量一直是一个无限接近 1 的数字。所以必然大于 0.6。**

---

`for` 循环里面还会对输出流的时间做限制，如下：

```
if (!check_recording_time(ost))
            return;
```

可以通过 `-t` 选项限制输出流的时间，其实限制输出流的时间在滤镜那里已经操作了，加了个 `trim` 滤镜，这里又用 `check_recording_time()` 可能是多余的。

---

接下来的重点是，把 `AVFrame` 往编码器发送，因为编码器可能会吐出来多个 `AVPacket`，所以需要用一个 `while(1){...}` 循环来接受，如下：

![1-7](do_video_out\1-7.png)

每次编码操作都会使用 `update_benchmark()` 函数来统计一下性能，这个函数特别好用，读者二开 ffmpeg.exe 的时候也可以使用 `update_benchmark()` 来统计性能。

从编码器拿到 `AVPacket` 之后，还需要转换一下时间基，如下：

```
av_packet_rescale_ts(pkt, enc->time_base, ost->mux_timebase);
```

最后调 `output_packet()` 写入文件即可，不过`output_packet()` 可能会先把 `AVPacket` 写入队列，而不是写到文件，因为必须等所有的输出流都初始化完成才能写入**头信息**，只有写入了头信息，才能开始写 `AVPacket`。

例如：现在编码出来一个视频帧 `AVPacket`，但是**音频输出流**还未初始化，所以头信息未写入，因此这个视频帧需要先写到队列缓存下来。

---

可以看到，如果不关注那 3 块复杂逻辑，`do_video_out()` 是比较简单的，就是从 buffer sink 读取 `AVFrame`，然后进行编码，转换时间基，写入文件就完事。

3 块复杂逻辑的分析，可以后续再看，如下：

1，《[FFmpeg可变帧率转恒定帧率详解](https://ffmpeg.xianwaizhiyin.net/ffmpeg/vfr_to_cfr.html)》

2，《[FFmpeg强制关键帧分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/force_keyframe.html)》

3，《[FFmpeg是如何调整帧率的](https://ffmpeg.xianwaizhiyin.net/ffmpeg/change_framerate.html)》

















