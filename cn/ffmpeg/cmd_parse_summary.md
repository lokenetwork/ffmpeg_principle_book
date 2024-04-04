# FFmpeg命令行参数解析总结—ffmpeg.c源码分析

<div id="meta-description---">命令行参数解析总结</div>

到这里，`ffmpeg.exe` 命令行的命令参数，已经全部解析到 `ffmpeg.c` 里面的结构体里面了，我们用一条命令来演示整个解析的流程，如下：

```
ffmpeg -i juren-5s.mp4 -f mp4 juren-5s-2.666
```

虽然我上面的输出文件的名称后缀是随便填的 666，但是使用了 `-f mp4`  选项强制指定封装格式为 mp4。

 `-f mp4` 的整个解析的过程是这样的。

**1，**通过  `split_command()` 函数把 `-f mp4`  解析到 `struct OptionGroup` 里面 `Option *opts` 的，如下：

```
typedef struct Option {
    const OptionDef  *opt; // ffmpeg_opt.c 里面的 options 变量定义的解析规则
    const char       *key; // f 
    const char       *val; // mp4
} Option;
```

**2，**再经过 `parse_optgroup()` 存到  `struct OptionsContex` 的 `format` 字段，如下：

![1-1](cmd_parse_summary\1-1.png)

**3，**最后这个 `mp4` 字符串，会丢进去 `avformat_alloc_output_context2()` 函数，如下：

![1-2](cmd_parse_summary\1-2.png)

上面的第一步，其实是把命令行参数解析成一个中间状态 `OptionParseContext`，经过  `parse_optgroup()` 函数处理之后，才会变成可以直接使用的 `OptionsContext` 结构。

---

最后，汇总一下命令行解析模块的全部流程 以及 数据结构。

![1-1](https://ffmpeg.xianwaizhiyin.net/ffmpeg/ffmpeg_parse_options/1-1.jpg)

#### **`split_commandline()` 函数的解析成中间状态 OptionParseContext ：**

![1-2](https://ffmpeg.xianwaizhiyin.net/ffmpeg/split_commandline/1-2.jpg)

**中间状态 OptionParseContext 关系图：**

![img](https://ffmpeg.xianwaizhiyin.net/ffmpeg/ffmpeg_parse_options/1-3.jpg)



#### `parse_optgroup()` 函数把中间状态 `OptionParseContext`  里面的 `OptionGroup` 转成 `OptionsContext` 的流程如下：

![1-3](https://ffmpeg.xianwaizhiyin.net/ffmpeg/open_files/1-3.jpg)

#### 输入输出流 与 输入输出滤镜之间的关联如下：

![0-3](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph/0-3.jpg)

---

命令行参数解析模块总结完毕。

最后补充一个知识点，你可以用 `-level debug` 把日志级别改成调试级别，就可以看到整个命令行参数的解析日志。如下：

```
ffmpeg -c:v h264 -i juren-5s.mp4 -f mp4 juren-5s-2.666 -y -level debug -hide_banner 1
```

```
av_log(NULL, AV_LOG_DEBUG, " matched as %s with argument '%s'.\n",
	groups[ret].name, arg);
```

