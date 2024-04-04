# parse_optgroup解析全局变量—ffmpeg.c源码分析

<div id="meta-description---">parse_optgroup解析全局变量</div>

`parse_optgroup()` 函数的定义如下：

```
/**
 * Parse an options group and write results into optctx.
 *
 * @param optctx an app-specific options context. NULL for global options group
 */
int parse_optgroup(void *optctx, OptionGroup *g);
```

注释写得非常清楚，就是如果 `optctx` 是 null ，就是解析到全局变量。如果 `optctx` 不是 null  ，就把 `OptionGroup *g` 解析到 `void *optctx` 里面。

这里提个醒，之前在 `split_commandline()` 里面也会解析到 `octx`，如下：

```
int split_commandline(OptionParseContext *octx, int argc, char *argv[],
                      const OptionDef *options,
                      const OptionGroupDef *groups, int nb_groups);
```

这两个变量跟结构体都非常相似，`octx` 与 `optctx`，`octx` 是 `OptionParseContext` 结构的，而 `optctx` 虽然是 `void*`，但是实际传的结构是 `OptionsContext`。

```
octx vs optctx
OptionParseContext vs OptionsContext
```

名字非常相似的，但是他们是不一样的。

---

我们先来讲一下 `parse_optgroup()` 解析全局变量的逻辑，如下：

![1-1](parse_optgroup\1-1.png)

可看到，比较简单，就是遍历数组 `g->opts`，然后调 `write_option()`。

实际上这个解析全局变量的逻辑，我们之前已经学过了，就是在《[FFplay是如何解析命令行参数的](https://ffmpeg.xianwaizhiyin.net/ffplay/parse_options.html)》一文中，虽然 `ffpaly` 播放器没有调  `parse_optgroup()` 函数，但是逻辑是类似的。

`ffpaly.exe` 是用 `find_option()` 函数找到命令行字符对应的 `OptionDef`，然后再传给 `write_option()` 处理。

`ffmpeg.exe` 是在调 `split_commandline()` 的时候已经把 命令行字符对应的 `OptionDef` 找到，并且保存进去 `OptionGroup` 结构里面，然后在 `parse_optgroup()` 里面再丢给 `write_option()` 处理。

`OptionDef`  是指上图代码里面的 `o->opt` 变量。

两者的流程图如下：

![1-2](parse_optgroup\1-2.jpg)

`ffmpeg.exe` 解析全局变量的流程就不画了，就一个循环里面调 `write_option()`，`OptionDef` 是已经事先准备好的，不需要再用 `find_option()` 函数去查。

---

`parse_optgroup()` +  `write_option()`  解析命令行参数 到 全局变量，是比较简单的，而且跟 `FFplay` 也非常类似。所以本文就到此结束了。

不过在 `open_files()` 的时候会再次调 `parse_optgroup()` 来解析参数，但是不是解析到全局变量，而是解析到 `OptionsContext` 结构体里面。

这个后面的文章继续分析。



