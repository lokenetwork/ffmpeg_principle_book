# FFmpeg与FFplay解析命令行的区别—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg与FFplay解析命令行的区别</div>

在前面的一章《[FFplay播放器分析](https://ffmpeg.xianwaizhiyin.net/ffplay/)》，讲过 `ffplay.exe` 命令行参数的处理。

其实 `ffmpeg.exe` 跟 `ffplay.exe` 处理命令行参数是有相似的逻辑的。

`ffmpeg.exe` 跟 `ffplay.exe` 都用了 `parse_option()` 来解析命令行参数，但是两者调用的用法有点奇怪。

回顾一下前文《[FFplay是如何解析命令行参数的](https://ffmpeg.xianwaizhiyin.net/ffplay/parse_options.html)》， `ffplay.exe` 处理命令行参数的流程，如下：

![1-3](ffplay_vs_ffmpeg\1-3.jpg)

可以看到，是从 `parse_option()` 里面调的 `write_option()`，而且在 `ffplay.exe` 里面  `write_option()` 的第一个参数 `optctx`，永远传的是 NULL。

下面是 `ffplay -x 400 -i juren.mp4` 命令的调试截图：

![1-4](ffplay_vs_ffmpeg\1-4.png)

上图中的 `func_arg` 函数 就是 `opt_width()`。

------

我们再来调试一下 `ffmpeg` ，命令如下：

```
ffmpeg -i juren.mp4 -vcodec h263 juren.flv
```

![1-5](ffplay_vs_ffmpeg\1-5.png)

上图是解析 `-vcodec h263` 选项的调试截图，上面的 `func_arg` 是 `opt_video_codec()` 函数。

可以看到，是从 `write_option()` 里面调的 `parse_option()`，然后 `parse_option()` 再调一次 `write_option()`，流程图如下：

![1-6](ffplay_vs_ffmpeg\1-6.jpg)

这个逻辑非常绕，我看了好几遍，用上调试器看函数调用才搞明白。

而且注意，`optctx` 不是 NULL，所以 `value` 会被保存进去 `optctx` 结构体的某个字段里面。

------

上面这些就是 `ffplay.exe` 与 `ffmpeg.exe` 调用 `parse_option()` 的区别。
