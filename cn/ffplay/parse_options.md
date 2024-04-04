# FFplay是如何解析命令行参数的？—ffplay.c源码分析

<div id="meta-description---">FFplay是如何解析命令行参数的</div>

为什么我会把命令行参数解析的文章放在后面，因为初学者使用 `FFplay`，通常不会去用太多的命令参数，通常初学者接触到的第一条 `FFplay` 命令是这样的。

```
ffplay -i juren.mp4
```

就是简简单单播放一个文件。

------

但是深入使用 `FFplay` 之后，必然会接触到各种各样的参数，你在工作中要验证一些问题的时候，也会用到各种各样的参数。因此理解 `FFplay` 命令行参数的解析逻辑是非常重要的。

命令行参数的解析逻辑，是通过一个 `OptionDef` 的数据结构定义的，如下：

```
typedef struct OptionDef {
    const char *name;
    int flags;
    .....
    union {
        void *dst_ptr;
        int (*func_arg)(void *, const char *, const char *);
        size_t off;
    } u;
    const char *help;
    const char *argname;
} OptionDef;
```

![1-1](parse_options\1-1.png)

`OptionDef` 其实是一个通用的数据结构，`ffprobe.exe` ，`ffplay.exe`  跟 `ffprobe.exe` 都是用的 `OptionDef` 数据结构来定义自己的命令行参数。

因此只要了解了 `OptionDef` 结构，基本上就成功了一半。

下面详细讲一下 `OptionDef` 里各个字段的含义。

1，`const char *name`，命令行参数名称，也就是 `-` 符号后面的字符串。

2，`int flags`，参数的属性，一共有 19 种属性，但是有些不是给 `ffpaly.exe` 用的，而是给 `ffmpeg.exe` 用的，等下会详细讲解各个值。

3，`union {...} u`，这是一个联合体，可以是参数的值，或者是处理参数的函数，又或者是 `OptionsContext` 结构体的偏移值（off），通过偏移值找到结构体中的某个字段。

在 `FFplay` 播放器里面，`union {...} u` 只会是 **参数的值** 或者 **是处理参数的函数**，不会用到偏移值（off），`off` 是给 `ffmpeg.exe` 用的。

4，`const char *help`，帮助信息。

5，`const char *argname`，参数的英文解释，用来拼接，输出帮助信息的。

------

现在来讲一下 **flags** 字段分别有哪些属性，这个 **flags** 字段是最重要的。

**1，**`#define HAS_ARG 0x0001`

代表命令行参数是否有值，例如 `-x 400` 指定播放宽度为 400 就是有值的，400 就是一个值。而 `-an` 代表不播放音频，`-an` 命令行参数是没有值的。

---

**2，**`#define OPT_BOOL 0x0002`

代表命令行参数是否是布尔值，布尔值后面都是没有 `value` 的，例如 `-an` 就是一个布尔值，如果想取反，需要在前面加上 `no` 前缀，例如 `-noan`

OPT_BOOL 跟 HAS_ARG 是相反的两个属性。

提醒：C99 标准是没有布尔值的，命令行的布尔参数最后会转成 0 跟 1 赋值给变量。

**3，**`#define OPT_STRING 0x0008`，代表命令行参数的值是一个字符串。

**4，**`#define OPT_INT 0x0080`，代表命令行参数的值是一个32位整数（int）。

**5，**`#define OPT_INT64 0x0400`，代表命令行参数的值是一个64位整数（int64）。

**6，**`#define OPT_FLOAT 0x0100`，代表命令行参数的值是一个单精度浮点数（float）。

**7，**`#define OPT_DOUBLE 0x20000`，代表命令行参数的值是双精度浮点数。

**8，**`#define OPT_TIME 0x10000`，代表命令行参数的值是时间字符串格式，如果格式不对会报错。

---

**9，**`#define OPT_EXPERT 0x0004`，只是一个标记，可以在 `show_help()` 输出帮助信息的时候指定 `flags` 为 `OPT_EXPERT` ，只显示这个标记相关的命令行参数。

**10，**`#define OPT_VIDEO 0x0010`，同  `OPT_EXPERT`，输出帮助信息用的。

**11，**`#define OPT_AUDIO 0x0020`，同  `OPT_EXPERT`，输出帮助信息用的。

**12，**`#define OPT_SUBTITLE 0x0200`，同  `OPT_EXPERT`，输出帮助信息用的。

**13，**`#define OPT_DATA 0x1000`，同  `OPT_EXPERT`，输出帮助信息用的。

**14，**`#define OPT_EXIT 0x0800`，这个属性代表处理完这个命令行参数的时候是否自动退出程序 ， 这个也是给 `show_help()` 用的，因为输入帮助信息之后，是需要退出程序的。

---

**15，**`#define OPT_PERFILE 0x2000`，这个属性目前只有 ffmpeg.exe 在用，因为 ffmpeg.exe 支持多个输入文件跟多个输出文件，所以这个属性是标记命令行参数只作用于一个输入/输出文件的。`OPT_PERFILE` 是跟 `OPT_OFFSET` 或者 `OPT_SPEC` 一起使用的。

**16，**`#define OPT_INPUT 0x40000`，代表命令行参数是作用于输入文件的，如果你把它用在输出文件前面，就会报错。

**17，**`#define OPT_OUTPUT0x80000`，代表命令行参数是作用于输出文件的，如果你把它用在输入文件前面，就会报错。

**18，**`#define OPT_OFFSET 0x4000 `，代表命令行参数的值会赋值给 `OptionsContext` 结构体的某个字段，这个字段的位置通过 偏移（u.off）找到。

**19，**`#define OPT_SPEC 0x8000` ，代表命令行参数的值会赋值给 `OptionsContext` 结构体里面的 `SpecifierOpt` 数组。

![1-2](parse_options\1-2.png)

上面的一级指针就是数组，然后下一个字段 `nb_codec_tags` 代表数组有多大。

因为 `SpecifierOpt`  是一个数组，所以有些命令行参数可以指定多个值，例如 

```
ffmpeg.exe -c:v h264_cuvid -c:v h264_mf -i juren.mp4 juren.flv
```

这样会优先使用 `h264_cuvid` 硬件解码器解码，如果 ffmpeg.exe 没有编译硬件解码器，就会使用 `h264_mf`

------

在 `ffplay.c` 里面，是调  `parse_options()` 函数来解析命令行参数的，流程图如下：

![0-1](parse_options\1-3.jpg)

其实命令行参数分为 2 种类型：

1，`ffplay.c` 里面定义的 `OptionDef options[] = {...}`，这些参数是 `ffplay` 播放器独有的。

2，组件库里面的参数。组件库是指，编解码库，容器库，重采样库，滤镜库等等。这些库支持的参数都可以在命令行里指定。

无论是通用的，还是私有的参数，都可以通过命令行参数指定，如下：

```
ffplay -probesize 1024 -export_all 1 -x 400 -i juren.mp4
```

上面的 `probesize` 是容器层**通用参数**，`mp4` 跟 `flv` 都有这个参数，但是 `export_all` 是 `mp4` 封装格式自己的**私有参数**。

------

下面来看一下 `parse_option()` 函数里面的重点代码，如下：

![1-4](parse_options\1-4.png)

可以看到，一开始就是从 `ffplay.c` 定义的 `options` 去找命令行参数是否存在，如果找不到，`po->name` 就是空。

如果命令行参数前面有 `no` ，那就把 `no` 去掉，再找一遍。

如果命令行参数是 布尔类型，那 `arg` 就会置为 1 或者 0。

------

当在 `ffplay.c` 的 `options` 找不到这个命令行参数的时候，就会找 `default` 的 `option`，在 `ffplay.c` 里面，`default option` 是一个函数调用。

![1-5](parse_options\1-5.png)

![1-6](parse_options\1-6.png)

`opt_default()` 函数，就是从各个组件库去找 **参数的定义**，如果也找不到，就会报错。

------

如果 从 `ffplay.c` 的 `optiions` 或者 组件库里面找到了命令行参数的定义，就会调 `write_option()` 来处理这个参数，重点如下：

![1-8](parse_options\1-8.png)

`write_option()` 里面的比较复杂的地方就是 `OPT_SPEC` 的处理逻辑，但是幸运的是，`ffplay` 播放器不会用到这块逻辑，所以我们暂时不需要关注它。

`OPT_SPEC` 是给 `ffmpeg.exe` 用的。

在 `ffplay` 播放器里面，`optctx` 参数总是 `NULL`，所以 `dst` 指针全都是指向一个全局变量的，不会指向 `OptionsContext` 结构体的某个字段。

当不能直接赋值给 `dst` 指针的时候，就会调 `u.func_arg` 函数来处理命令行参数，**这样设计让命令行解析更加灵活**。

最后，如果这个命令行参数的属性标记如果是 `OPT_EXIT`，那解析完这个参数之后，就会立即退出程序。`OPT_EXIT` 通常是给帮助参数用的，因为输出帮助信息到控制台之后，确实是需要立即退出程序了。

---

总结，`ffplay.c` 解析命令行参数，只有两种逻辑，**一是**直接赋值给全局变量，**二是**调  `u.func_arg` 函数来处理。

`ffprobe.exe` 跟 `ffplay.exe` 的命令行参数解析逻辑是一样的，都是用的  `parse_options()`。

但是 `ffmpeg.exe` 不是，`ffmpeg.exe` 用的是 `ffmpeg_parse_options()` 函数，稍微更复杂一些。

但是 `parse_options()` 跟  `ffmpeg_parse_options()`  内部都用了  `parse_option()`  。

注意  `parse_options()` 跟  `parse_option()` ，一个有 s ，一个没 s。

