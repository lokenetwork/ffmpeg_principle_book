# FFmpeg命令行参数分析-ac—ffmpeg.c源码分析

<div id="meta-description---">ac 参数的全称是 audio channels（声道数量），可以通过这个参数指定 输入 或者 输出 的声道数量</div>

`ac` 参数的全称是 audio channels（声道数量），可以通过这个参数指定 输入 或者 输出 的声道数量，定义如下：

![1-1](cmd_arg_ac\1-1.png)

本文的素材可以在 百度网盘 进行下载。

---

#### ac 参数作用于输入源

当 `ac` 参数作用于输入源的时候，通常是因为输入源是 pcm 数据，pcm 格式是没有头部记录这个文件是多少声道的，所以你需要指定声道数量才能正确解析输入源，具体的命令如下：

```
ffmpeg -ar 48000 -ac 2 -f s16le -i juren-5s.pcm -f mp4 juren-5s.mp4
```

当 `ac` 作用于输入源的时候，它的实现原理如下：

1. 把 `ac` 参数的值赋值到 `OptionsContext` 结构的 `audio_channels` 字段
2. 把 `audio_channels` 赋值到 `o->g->format_opts`
3. 把 `o->g->format_opts` 传递给 `avformat_open_input()` 函数打开输入源

代码如下：

![1-2](cmd_arg_ac\1-2.png)

![1-3](cmd_arg_ac\1-3.png)

---

#### ac 参数作用于输出源

当 `ac` 参数作用于输出源的时候，它的作用是 对声道数量进行转换，例如把输入的 2 声道 转换成 单声道输出，命令如下：

```
ffmpeg -i juren-5s.mp3 -ac 2 juren-5s-1.mp3
```

`juren-5s.mp3` 是 2 声道的音频，而 `juren-5s-1.mp3` 是 单声道的音频。

当 `ac` 作用于输出源的时候，它的实现原理如下：

**1，**把 `ac` 参数的值赋值到 `OptionsContext` 结构的 `audio_channels` 字段

**2，**把 `audio_channels` 赋值给编码器参数，因为编码器需要知道自己编码的音频数据是多少声道的，代码如下：

![1-4](cmd_arg_ac\1-4.png)

**3，**根据编码器参数 `enc_ctx->channels` 来设置 `OutputFilter` 的 `channel_layout`（声道布局），如下：

![1-5](cmd_arg_ac\1-5.png)

注意：上图这个 `OutputFilter` 里面会是一个 `buffersink` 的 滤镜，用来连接输出流 `OutputStream` 的。

---

**4，**当 `OutputFilter` 的 `channel_layout` 被设置的时候，就会创建 `aformat` 滤镜来进行声道转换，如下：

![1-6](cmd_arg_ac\1-6.png)

滤镜里面最后设置的声道布局，而不是声道数量，我估计是因为声道布局更准确，因为知道声道布局，自然就知道声道数量了。

---

**5，**用 `buffersink` 的输出声道信息重新设置编码器参数，如下：

![1-7](cmd_arg_ac\1-7.png)

这是比较绕的一步，因为前面已经设置过了 编码器参数，这里又设置一遍，是为什么呢？

我估计是这样的，因为 `-complex_filter` 或者 `-af` 命令行参数也可以直接指定 `aformat` 滤镜，这个有可能会把 `-ac` 的效果覆盖掉。

所以编码器最终的声道数是通过 `buffersink` 来决定的，这是合理的，因为编码器的数据，就是由 `buffersink` 来输入的。

