# FFmpeg命令行参数分析-b:v—ffmpeg.c源码分析

<div id="meta-description---">b:v 参数中的 b 代表 bitrate，v 代表 video ，组合起来就是设置视频流的码率</div>

`b:v` 参数中的 `b` 代表 `bitrate`，`v` 代表 `video`，组合起来就是设置视频流的码率，用法如下：

```
ffmpeg -i juren-30s.mp4 -b:v 200k output.flv
```

---

下面来介绍一下 `ffmpeg.c` 里面是怎么实现 `b:v` 功能的。

这种带 `:` 冒号的参数是第一次讲，实际上在 `ffmpeg_opt.c` 的 `options[]` 数组变量里是找不到 `b:v` 的定义的。

那 `ffmpeg.c` 是怎么解析  `b:v`  的呢？我们可以断点调试一下 `split_commandline` 函数，如下：

![1-1](cmd_arg_bv\1-1.png)

调试之后发现，是 `find_option()` 函数做了一些处理，如下：

![1-2](cmd_arg_bv\1-2.png)

它判断了 `b:v` 这个字符串的开头的地方，所以实际上 `b:v` 等于 `b`，如下：

![1-3](cmd_arg_bv\1-3.png)

提示：`b:v` 的属性是 `OPT_OUTPUT`，所以它是一个只能用于输出源的参数。

下面这两条命令是等价的。

```
ffmpeg -i juren-30s.mp4 -b:v 200k output.flv
ffmpeg -i juren-30s.mp4 -b 200k output.flv
```

---

了解完 `ffmpeg.c` 是怎么解析 `b:v` 后，我们再来看一下它是怎么使用 200k 这个值的。如下：

**1，**首先 `opt_bitrate()` 函数会把 200k 设置给 `o->g->codec_opts`，如下：

![1-4](cmd_arg_bv\1-4.png)

**2，**`codec_opts` 被设置到 `ost->encoder_opts` ，如下：

![1-5](cmd_arg_bv\1-5.png)

`filter_codec_opts()` 函数会对 `codec_opts` 进行过滤的，只提取符合条件的编码器参数设置到 `ost->encoder_opts`  

---

**3，**把 `ost->encoder_opts` 参数传递给 `avcodec_open2()` 函数，如下：

![1-6](cmd_arg_bv\1-6.png)

通过这个例子，就如何调 API 设置编码器的码率了。类似的参数还有 `b:a`，代表音频流的码率。
