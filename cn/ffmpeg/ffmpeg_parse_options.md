# ffmpeg_parse_options命令行解析—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg_parse_options源码分析</div>

`ffmpeg_parse_options()` 是 `ffmpeg.exe` 解析命令行参数的**主函数**，它的流程图如下：

![1-1](ffmpeg_parse_options\1-1.jpg)

`main()` 在调 `ffmpeg_parse_options()` 之前还有一些初始化的准备工作，不过不是重点。

从上图可以看到，`ffmpeg_parse_options()` 内部有 4 个重要的函数调用，如下：

**1，**`split_commandline()`，作用是把命令行的参数先解析到一个中间结构（OptionParseContext）里面，这个中间结构就是 `octx` 变量，

**2，**`parse_optgroup(NULL, &octx.global_opts)`，把 `octx` 里面的 `global_opts` 字段解析到**全局变量**。

**3，**`open_files(&octx.groups[GROUP_INFILE],...)`，`octx.groups[GROUP_INFILE]` 里面会有一个或者多个输入文件，需要用这些信息打开一个或者多个输入文件。

**4，**`open_files(&octx.groups[GROUP_OUTFILE],...)`，`octx.groups[GROUP_OUTFILE]` 里面会有一个或者多个输出文件，需要用这些信息打开一个或者多个输出文件。

------

命令行参数解析模块，主要用到 3 个数据结构，如下：

**1，**`struct OptionParseContext`，存储命令行参数的中间状态。定义如下：

```
typedef struct OptionParseContext {
    OptionGroup global_opts;

    OptionGroupList *groups;
    int           nb_groups;

    /* parsing state */
    OptionGroup cur_group;
} OptionParseContext;
```

`ffmpeg.exe` 在使用 `OptionParseContext` 结构的时候，只定义了两个组，输入组跟输出组，如下：

```
static const OptionGroupDef groups[] = {
    [GROUP_OUTFILE] = { "output url",  NULL, OPT_OUTPUT },
    [GROUP_INFILE]  = { "input url",   "i",  OPT_INPUT },
};
//初始化 octx
init_parse_context(octx, groups, nb_groups);
```

`ffmpeg.exe` 初始化 `octx` 变量是在 `init_parse_context()` 函数里面的，这个比较简单，不过多介绍。

**输入组 里面是可以有多个输入文件的，输出组也可以有多个输出文件。**

------

**2，**`struct OptionGroup`，存储单个输入或者输出文件的信息。

```
typedef struct OptionGroup {
    const OptionGroupDef *group_def;
    const char *arg; //这个字段是输入或者输出的文件名

    Option *opts; // ffmpeg_opt.c 里面定义的选项
    int  nb_opts;

    AVDictionary *codec_opts;  //编码器选项
    AVDictionary *format_opts; //容器选项
    AVDictionary *resample_opts;  //重采样选项 应该是快废弃的
    AVDictionary *sws_dict; //图像转换选项
    AVDictionary *swr_opts; //重采样选项
} OptionGroup;
```

`OptionGroup` 是一个非常重要的数据结构，存储单个输入或者输出文件的命令行参数信息，之前在《[ffmpeg命令参数类型](https://ffmpeg.xianwaizhiyin.net/base-ffmpeg/ffmpeg-cmd-type.html)》讲过，每个输出文件都可以有自己的独立参数，不同的输出文件的参数是分开的。不同的输出文件可以指定不同的编码器等等。

`codec_opts`，`format_opts`，`sws_dict`，`swr_opts` 这些字段存储的是**组件库的参数**。

`Option *opts` 字段存储的才是 `ffmpeg.exe` 工具自身的参数。

---

`OptionGroup` 里面的 `arg` 是文件名，然后 `opts` 存储的是 `ffmpeg_opt.c` 里面的 `options[]` 变量中的某些选项，如下：

```
typedef struct Option {
    const OptionDef  *opt; //ffmpeg_opt.c 里面的选项
    const char       *key; //命令行参数的 key
    const char       *val; //命令行参数的 value
} Option;
```

![1-2](ffmpeg_parse_options\1-2.png)

如果你命令行使用了 `-f 888`，那 `Option` 的内容就是下面这样的：

```
const OptionDef  *opt = options[1] // f 选项的下标是 1
const char       *key = f 
const char       *val = 888
```

`OptionDef` 结构在分析 `ffplay` 的时候讲过了，推荐阅读《[FFplay是如何解析命令行参数的](https://ffmpeg.xianwaizhiyin.net/ffplay/parse_options.html)》。

`ffplay.exe` 跟 `ffmpeg.exe` 都是用的 `OptionDef` 结构来定义自身支持的参数。

---

**3，**`struct OptionGroupList`，`OptionGroup` 的列表。

```
typedef struct OptionGroupList {
    const OptionGroupDef *group_def;

    OptionGroup *groups;
    int       nb_groups;
} OptionGroupList;
```

```
static const OptionGroupDef groups[] = {
    [GROUP_OUTFILE] = { "output url",  NULL, OPT_OUTPUT },
    [GROUP_INFILE]  = { "input url",   "i",  OPT_INPUT },
};
```

通常 `OptionGroup` 裸露在外面直接用，例如 `OptionParseContext::global_opts`，那就是全局组，如果 `OptionGroup` 包在 `OptionGroupList` 使用的，`OptionGroup` 的属性就由 `OptionGroupDef` 确定，确定他是输入组还是输出组。

---

上面这些数据结构的关系如下图所示：

![1-3](ffmpeg_parse_options\1-3.jpg)

不过 `OptionGroup` 跟 `OptionGroupList` 的结构设计得有点奇怪，他们里面都有一个 `OptionGroupDef`。

全局组的定义如下：

```
static const OptionGroupDef global_group = { "global" };
```

![1-4](ffmpeg_parse_options\1-4.png)

------

相关的数据结构已经了解了，后面的文章会继续分析 `ffmpeg_parse_options()` 里面各个子函数的内部逻辑。
