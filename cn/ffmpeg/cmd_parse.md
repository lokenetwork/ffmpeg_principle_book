# FFmpeg命令行参数解析—ffmpeg.c源码分析

<div id="meta-description---">命令行参数解析模块，是 ffmpeg.exe 转换器里面最复杂的模块之一，但是正是由于它的复杂，才支撑起来命令行那些强大的语法。</div>

命令行参数解析模块，是 `ffmpeg.exe` 转换器里面最复杂的模块之一，但是正是由于它的复杂，才支撑起来命令行那些强大的语法。

在学习 `ffmpeg.c` 里面的命令行模块的时候，可以把 日志级别 `loglevel` 设置为 `debug`，如下：

```
ffmpeg -hide_banner -re -i juren.mp4 -t 5 juren-5s.mp4 -loglevel debug
```

![1-1](cmd_parse\1-1.png)

这样就能看到每个命令行参数的解析过程。

---

命令行参数解析模块的逻辑是在 `ffmpeg_opt.c` 文件里面，这个文件有 6 个比较重要的函数调用，如下：

**1，**[ffmpeg_parse_options()](https://ffmpeg.xianwaizhiyin.net/ffmpeg/ffmpeg_parse_options.html)，解析命令行参数的**主函数**。

**2，**[split_commandline()](https://ffmpeg.xianwaizhiyin.net/ffmpeg/split_commandline.html)，把命令行的参数先解析到一个中间结构（OptionParseContext）里面

**3，**[parse_optgroup(NULL, &octx.global_opts)](https://ffmpeg.xianwaizhiyin.net/ffmpeg/parse_optgroup.html)，把 `OptionParseContext` 里面的 `global_opts` 解析到全局变量。

**4，**[open_files(&octx.groups[GROUP_INFILE],...)](https://ffmpeg.xianwaizhiyin.net/ffmpeg/open_files.html)，`octx.groups[GROUP_INFILE]` 里面会有一个或者多个输入文件，需要用这些信息打开一个或者多个输入文件。

**5，**[open_files(&octx.groups[GROUP_OUTFILE],...)](https://ffmpeg.xianwaizhiyin.net/ffmpeg/open_files.html)，`octx.groups[GROUP_OUTFILE]` 里面会有一个或者多个输出文件，需要用这些信息打开一个或者多个输出文件。

**6，**[init_simple_filtergraph初始化简单滤镜](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph.html)， `InputFilter`，`OutputFilter` 滤镜是连接 输入流 与 输出流 的桥梁。

阅读本章节之前，推荐回顾一遍《FFmpeg基础》里面的《[ffmpeg命令参数类型](https://ffmpeg.xianwaizhiyin.net/base-ffmpeg/ffmpeg-cmd-type.html)》一文。

