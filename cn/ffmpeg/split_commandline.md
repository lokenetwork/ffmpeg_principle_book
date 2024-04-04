# split_commandline解析中间状态—ffmpeg.c源码分析

<div id="meta-description---">split_commandline解析中间状态</div>

`split_commandline()` 函数的作用是把命令行的参数先解析到一个中间结构（OptionParseContext）里面，也就是 `octx` 变量。

本文我们一起来探索一下它是如何实现解析功能的，`split_commandline()` 函数的定义如下：

```
/**
 * Split the commandline into an intermediate form convenient for further
 * processing.
 *
 * The commandline is assumed to be composed of options which either belong to a
 * group (those with OPT_SPEC, OPT_OFFSET or OPT_PERFILE) or are global
 * (everything else).
 *
 * A group (defined by an OptionGroupDef struct) is a sequence of options
 * terminated by either a group separator option (e.g. -i) or a parameter that
 * is not an option (doesn't start with -). A group without a separator option
 * must always be first in the supplied groups list.
 *
 * All options within the same group are stored in one OptionGroup struct in an
 * OptionGroupList, all groups with the same group definition are stored in one
 * OptionGroupList in OptionParseContext.groups. The order of group lists is the
 * same as the order of group definitions.
 */
int split_commandline(OptionParseContext *octx, int argc, char *argv[],
                      const OptionDef *options,
                      const OptionGroupDef *groups, int nb_groups);
```

在 `ffmpeg_opt.c` 里面的实际传参如下：

```
static const OptionGroupDef groups[] = {
    [GROUP_OUTFILE] = { "output url",  NULL, OPT_OUTPUT },
    [GROUP_INFILE]  = { "input url",   "i",  OPT_INPUT },
};
```

![1-1](split_commandline\1-1.png)

------

`split_commandline()` 函数的内部流程图如下：

![1-2](split_commandline\1-2.jpg)

`split_commandline()` 函数的整体逻辑是这样的。首先是一个 `while (optindex < argc) {...}` 循环，不断处理命令行的参数。

在循环内部，会多次查找 **命令行的参数** 所对应的 `struct OptionDef` ，`f` 参数对应的 `OptionDef` ，如下：

![1-2](https://ffmpeg.xianwaizhiyin.net/ffmpeg/ffmpeg_parse_options/1-2.png)

查询 `OptionDef` 会最多会尝试 3 次：

**1，**从 `ffmpeg_opt.c` 定义的 `options` 数组里面找，这是 `ffmpeg.exe` 独有的 `OptionDef` （选项）。

**2，**从 编解码库，容器库，重采样库，等等里面查找公共的 `OptionDef` 。调用的函数是 `opt_default()`。

**3，**判断命令行参数，是否以 no 开头的，例如 `nobenchmark` 会转成 `benchmark` ，然后再从 `ffmpeg_opt.c` 定义的 `options` 数组里面找一次。

这是 FFmpeg 整个项目布尔参数的习惯，如果要取反，就在前面加个 `no`。解析的时候会把  `no` 去掉，然后赋值为 0 ，如下：

```
/* boolean -nofoo options */
if (opt[0] == 'n' && opt[1] == 'o' &&
    (po = find_option(options, opt + 2)) &&
    po->name && po->flags & OPT_BOOL) {
    add_opt(octx, po, opt, "0"); //注意这里的 “0”
    av_log(NULL, AV_LOG_DEBUG, " matched as option '%s' (%s) with "
           "argument 0.\n", po->name, po->help);
    continue;
}
```

------

一旦找到 **命令行的参数** 所对应的 `struct OptionDef` ，就会开始记录。

如果是在 `ffmpeg.exe` 的**独立参数**里面找到的 `OptionDef` ，可能会记录到下面两个地方：

**1，**`octx->global_opts`，记录到全局组里面。

**2，**`octx->cur_group`，记录到当前组里面。

![1-3](split_commandline\1-3.png)

如果是在编解码库，容器库等等，公共的组件库里面里面找到的 `OptionDef` ，可能会记录到下面这些地方，是通过 `opt_default()` 函数记录的。

**1，**`codec_opts`，编解码器参数

**2，**`format_opts`，容器层参数。

**3，**`sws_dict`，图像转换

**4，**`swr_opts`，重采样

无论是 `octx->cur_group` 当前组，还是 `codec_opts`，`format_opts` 等等，他们都是一种中间的状态，当匹配到输入或者输出文件的时候，就会把这些中间信息，全部转移到输入组，或者是输出组里面。

这个转移操作，是通过 `finish_group()` 函数完成的，如下：

![1-4](split_commandline\1-4.png)

---

接下来讲一下，ffmpeg.exe 是怎么匹配到输入 跟 输出文件的。

**输入文件的匹配代码如下：**

```
/* named group separators, e.g. -i */
if ((ret = match_group_separator(groups, nb_groups, opt)) >= 0) {
	GET_ARG(arg);
	finish_group(octx, ret, arg);
	av_log(NULL, AV_LOG_DEBUG, " matched as %s with argument '%s'.\n",
		groups[ret].name, arg);
	continue;
}
```

上面的 `groups` 变量的定义如下：

```
static const OptionGroupDef groups[] = {
    [GROUP_OUTFILE] = { "output url",  NULL, OPT_OUTPUT },
    [GROUP_INFILE]  = { "input url",   "i",  OPT_INPUT },
};
```

而 opt 就是命令行字符。

所以逻辑比较简单，就是匹配 opt 是不是 i ，如果是 i， 就用 `GET_ARG(arg)` 获取输入文件的名称，存到 `arg` 里面，然后调 `finish_group()` 完成输入组的解析。

**输出文件的匹配代码如下：**

```
/* unnamed group separators, e.g. output filename */
if (opt[0] != '-' || !opt[1] || dashdash+1 == optindex) {
    finish_group(octx, 0, opt);
    av_log(NULL, AV_LOG_DEBUG, " matched as %s.\n", groups[0].name);
    continue;
}
```

上面的 `finish_group(octx, 0, opt);` 里面的 0 应该替换成 `GROUP_OUTFILE` 宏，这样看起来会更容易理解。

上面的代码前面两个条件比较简单，就是匹配命令行参数，`opt` 的第一个字符如果不是 `-` ，而且第二个字符没有值，这个 `opt` 变量就是输出文件的文件名。

举个例子参数 "juren.mp4" ，就符合上面条件的判断。

不过有个 dashdash 变量不太容易看懂，这其实是一个**转义操作**，如果你的输出文件名命名，前面确实要是 `-` ，那就可以用 `--` 来转义，如下：

```
ffmpeg -i juren.mp4 -- -juren.flv
```

上面最后一个条件，`dashdash+1 == optindex`，的意思是，只要匹配到 `--`，那后面的参数就是输出文件名称。

其他命令也有类似习惯，比如 `grep` 搜索带 `-` 的字符串。

```
grep -r "-h" 
```

上面这样是不行的，`-h` 会当成 `grep` 自己的参数配置，需要换成下面的命令。

```
grep -r -- "-h" 
```

---

之前说过，一个输入组是可以有多个输入文件的，输出组同理。所以每调用一次 `finish_group()` ，它的内部都会扩展一次数组的大小，来容纳更多的输入文件，如下：

![1-5](split_commandline\1-5.png)

------

`split_commandline()` 函数执行完毕之后，所有的命令行参数都会解析进去中间结构体 `OptionParseContext` 里面。

`OptionParseContext` 里面有全局参数组，输入组，输出组。

理解了 `split_commandline()` 函数，也就理解了命令行参数解析模块的50%。

---

题外话：

`opt_default()` 函数是处理公共组件库的参数解析的，会把命令行参数赋值到 `codec_opts`，`format_opts`，`sws_dict`，`swr_opts`。定义如下：

```
int opt_default(void *optctx, const char *opt, const char *arg)
```

但是 `opt_default()` 内部是没用使用到 第一个参数 `void *optctx` 的，**所以第一个参数是多余的**。
