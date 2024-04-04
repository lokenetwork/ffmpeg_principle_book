# ffplay.c源码分析

<div id="meta-description---"> FFplay 是 FFmpeg 官方提供的一个播放器的实现，全部的逻辑代码都在 ffplay.c 里面，只有不到 4 千行代码，麻雀虽小，五脏俱全。</div>

`FFplay` 是 `FFmpeg` 官方提供的一个播放器的实现，全部的逻辑代码都在 `ffplay.c` 里面，只有不到 4 千行代码，麻雀虽小，五脏俱全。

`FFplay` 播放器支持大部分常见的播放器功能，例如 快进快退，逐帧播放，滤镜。

可以通过以下命令查看 ffplay 播放器支持的所有功能。

```
ffplay --help
```

![1-1](index\1-1.png)

下面就让我们通过`clion`调试，一起探索 `FFplay` 播放器的实现。

`clion` 调试 `ffplay` 可以阅读《[用Ubuntu18与clion调试FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html)》一文。

