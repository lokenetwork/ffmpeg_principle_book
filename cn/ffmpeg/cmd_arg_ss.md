# FFmpeg命令行参数分析-ss—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg 可以通过 ss 参数设置起始时间，那这个功能是怎么实现的？</div>

`ss` 参数的全称是 set the start time（设置从哪里开始），定义如下：

```
{"ss", HAS_ARG | OPT_TIME | OPT_OFFSET | OPT_INPUT | OPT_OUTPUT, { .off = OFFSET(start_time) }, "set the start time offset", "time_off"},
```

从 `OPT_INPUT` 与 `OPT_OUTPUT` 两个属性可以知道，`ss` 参数可以作用于输入文件，也可以作用于输出文件。

---

#### ss 参数作用于输入文件

`ss` 参数用于输入文件的命令如下：

```
ffmpeg -an -ss 10 -i juren.mp4 juren.flv
```

上面这条命令的作用是跳转到 `juren.mp4` 第 10 秒的位置开始处理数据，在 `ffmpeg.c` 里面的实现流程如下：

**1，**`ss` 参数会赋值到 `OptionsContext` 的 `start_time` 字段。这是常规操作了，大部分参数都是解析到  `OptionsContext` 的某个字段的。

**2，**`start_time` 变量经过处理，转换时间单位 之后，会传递给 `avformat_seek_file()` 函数进行跳转，如下：

![1-1](cmd_arg_ss\1-1.png)

上图还处理了 `AVFMT_SEEK_TO_PTS` 的情况，可以看到 FFmpeg 里面其实处理了非常多的兼容性问题。对不同的格式都进行了处理，自身项目如果要调 `avformat_seek_file` 这个函数，最好把上面的 `AVFMT_SEEK_TO_PTS` 判断也照抄进去项目。

---

#### ss 参数作用于输出文件

`ss` 参数用于输出文件的命令如下：

```
ffmpeg -an -i juren.mp4 -ss 10 juren.flv
```

上面这条命令的作用是 裁剪输出文件，把前面的10秒数据丢弃。在 `ffmpeg.c` 里面的实现流程如下：

**1，**`ss` 参数会赋值到 `OptionsContext` 的 `start_time` 字段。常规操作

**2，**`OptionsContext::start_time` 会再次赋值到 `OutputFIle` 的 `start_time`，如下：

```
of->start_time = o->start_time;
```

那 `of->start_time` 又是在哪里被使用的呢？

答：在 `configure_output_video_filter()` 与  `configure_output_audio_filter()` 配置滤镜的时候，如下：

![1-2](cmd_arg_ss\1-2.png)

可以看到，插入了 `trim` 滤镜，裁剪功能就是用 `trim` 实现的。

----

总结，当 `ss` 参数作用于输入文件的时候，是通过 `avformat_seek_file()` 函数实现的。而当 `ss` 参数作用于输出文件的时候，是通过 `trim` 滤镜实现裁剪的功能。

------

TODO：`adjust_frame_pts_to_encoder_tb` 函数好像也用到了 `of->start_time `，后面补充讲解。

