# FFmpeg命令行参数分析-vframes—ffmpeg.c源码分析

<div id="meta-description---">vframes 的作用是限制输出源的视频帧数量</div>

`vframes` 参数的全称是 video frame，作用是限制输出源的视频帧数量，用法如下：

```
ffmpeg -i juren.mp4 -an -vframes 10 juren.flv
```

上面的命令只提取了 10帧视频保存到 `flv` 里面。

`vframes` 参数定义如下：

![1-1](cmd_arg_vframes\1-1.png)

上图中，`vframes` 参数有`OPT_VIDEO` 与 `OPT_OUTPUT` 两个属性，所以它是作用于输出的参数，只对视频生效。

定义中，会调 `opt_video_frames()` 函数来解析 `vframes` 参数，如下：

```
static int opt_video_frames(void *optctx, const char *opt, const char *arg)
{
    OptionsContext *o = optctx;
    return parse_option(o, "frames:v", arg, options);
}
```

在 `opt_video_frames()` 里面，把 `vframes` 换成了 `frames:v`，所以实际上解析的是  `frames:v`，这种带冒号 `:` 的解析方法之前在 《[FFmpeg命令参数分析-b:v](https://ffmpeg.xianwaizhiyin.net/ffmpeg/cmd_arg_bv.html)》讲过了，只会匹配冒号前面的部分，所以真正的定义是 `frames`，如下：

![1-2](cmd_arg_vframes\1-2.png)

---

首先，`vframes` 参数会赋值给 `OptionsContext` 结构的 `max_frames` 字段。

然后 `OptionsContext::max_frames` 会赋值给 `ost->max_frames` ，如下：

![1-3](cmd_arg_vframes\1-3.png)

#### 那 ost->max_frames 会在那些地方被使用呢？

#### 1，do_video_out 函数

在《[do_video_out视频编码封装](https://ffmpeg.xianwaizhiyin.net/ffmpeg/do_video_out.html)》一文简单讲解过 `do_video_out` 函数的逻辑。

![1-4](cmd_arg_vframes\1-4.png)

#### 2，need_output 函数

在 `need_output()` 函数里面，当达到最大的视频帧数的时候，就会把 视频流关联的输入文件里面的所有流都关闭，包括音频流，注意下图中的 for 循环。

![1-5](cmd_arg_vframes\1-5.png)

---

至此，`vframes` 的实现原理就介绍完毕了，不过 `vframes` 有另一种用法，可以实现任意时间抽帧，命令如下：

```
ffmpeg -i juren.mp4 -ss 00:00:05.000 -vframes 1 out.png
```

上面这条命令用了 `-ss` 跳转到第 5 秒的位置，然后只输出一帧保存成 png，就立即退出了。
