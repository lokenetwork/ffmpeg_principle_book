# open_files打开输入输出文件—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

在 `ffmpeg.exe` 里面打开输入输出文件，是由 `open_files()` 函数来完成的，如下：

![1-1](open_files\1-1.png)

`open_files()` 函数的定义如下：

```
static int open_files(OptionGroupList *l, const char *inout,
                      int (*open_file)(OptionsContext*, const char*))
```

参数解析如下：

**1，**`OptionGroupList *l`，输入/输出文件信息列表，

`octx.groups[GROUP_INFILE]`，`octx.groups[GROUP_OUTFILE]`，`OptionParseContext octx` 是在 `split_commandline()` 里面提前处理好的。

**2，**`const char *inout`，标记字符，用来输出日志的。

**3，**`int (*open_file)(OptionsContext*, const char*)`，函数指针，指向真正的打开输入文件函数，或者是打开输出文件的函数。

---

`open_files()` 函数的流程如下：

![1-2](open_files\1-2.jpg)

`open_files()` 函数的逻辑就是 `for` 循环遍历 `OptionGroupList` 列表，`OptionGroupList->nb_groups` 就是文件的数量，如果命令行指定有 3 个输入文件，`OptionGroupList->nb_groups` 就等于 3

`init_options()` 函数是 初始化 `OptionsContext o`  的，设置一些字段的默认值。

然后调 `parse_optgroup()` 函数把 `OptionGroup g` 里面的参数解析到 `OptionsContext o` 结构，然后用 `open_file()` 函数打开文件。

`open_file` 是一个函数指针，可能指向 `open_input_file()` ，也可能指向 `open_output_file()`。

---

我们需要先学习一下 `struct OptionsContext` 的结构，`OptionsContext` 保存的是单个输入或者输出文件的信息。

据本人统计，`OptionsContext` 里面的字段有50个以上，非常庞大，如下：

```
typedef struct OptionsContext {
    OptionGroup *g;

    /* input/output options */
    int64_t start_time;
    int64_t start_time_eof;
    int seek_timestamp;
    const char *format;

    SpecifierOpt *codec_names;
    int        nb_codec_names;
    SpecifierOpt *audio_channels;
    int        nb_audio_channels;
    ...省略...
}
```

这 50 多个字段可以分为 3 种类型。

**1，**`OptionGroup *g`，`OptionsContext` 本身就是从 `OptionGroup` 里面的 `Option` 解析过来，这里又保存一下 `OptionGroup` ，是因为要用到里面的 `codec_opts` ，`format_opts`，这些组件库相关的参数，是没有解析到 `OptionsContext` 里面的。

**2，**`int64_t start_time` ，`int seek_timestamp`，`const char *format`，这些，我把他们称为**单字段**，都是普通类型。

**3，**`SpecifierOpt`，**数组字段**，这是最重要，也是比较复杂的字段。数组字段是一个数组，可以有多个的。

------

`parse_optgroup()` 函数把 `OptionGroup` 转成 `OptionsContext` 的流程如下：

![1-3](open_files\1-3.jpg)

可以看到，`Option` 里面原本都是 key - value 的字符串结构，经过 `parse_optgroup()` 处理之后，就变成了，右边那种 `int64` ，`int`，`SpecifierOpt` 的结构。

至于 `AVDictionary` 相关的结构是没有进行转换的，因为 `AVDictionary` 可以直接传给 FFmpeg 的 API 函数，所以 **open_files()** 会把 `OptionGroup` 直接赋值给 `OptionsContext` 的 `g` 字段，通过 `OptionsContext->g->codec_opts` 的方式，即可提取命令行指定的编解码参数。

![1-4](open_files\1-4.png)

---

下面来具体分析一下 `parse_optgroup()` 函数是如何把  `OptionGroup` 里面的 key - value 转成 `OptionsContext` 特定的字段的。

不过与其说是  `parse_optgroup()` 做的转换，不如说是 `write_option()` 函数做的转换。

 `parse_optgroup()` 的逻辑比较简单，只是在循环里面调了一下  `write_option()` ，如下：

![1-5](open_files\1-5.png)

---

下面来分析一下 `write_option()`  是如何做转换的，定义如下：

```
static int write_option(void *optctx, const OptionDef *po, const char *opt,
                        const char *arg)
```

`write_option()` 的**第一步是确定目标位置**，这个 `key-value` 要保存到哪里，如下：

![1-6](open_files\1-6.png)

可以看到，是通过 flags 标记来判断要保存到全局变量，还是保存到 `OptionsContext` 结构，全局变量的解析流程之前已经讲过了，本文主要关注 解析到 `OptionsContext` 结构的流程。

可以看到通过 `optctx + po->u.off` 就能定位到 `OptionsContext` 里面的的字段，这个 `u.off` 是通过一个宏计算出来的，如下：

```
#define OFFSET(x) offsetof(OptionsContext, x)
```

![1-7](open_files\1-7.png)

`OptionsContext` 里面的 `int`，`int64`，`double`，等等字段的更新都比较简单，就是通过偏移定位到字段，然后赋值就行了。

但是 `OptionsContext` 里面 `SpecifierOpt` 类型的字段赋值比较复杂。`SpecifierOpt` 结构体的定义如下：

```
typedef struct SpecifierOpt {
    char *specifier;    /**< stream/chapter/program/... specifier */
    union {
        uint8_t *str;
        int        i;
        int64_t  i64;
        uint64_t ui64;
        float      f;
        double   dbl;
    } u;
} SpecifierOpt;
```

可以看到 `SpecifierOpt` 只有两个字段，

**1，**`char *specifier`，会是 `a` 或者 `v`，区分这是音频还是视频的选项，或者留空字符，代表不区分音视频。

**2，**`union {...} u`，一个联合体，用来适应各种各样的数据类型。`uint8_t` 实际上是字符串，因为一个 `char` 也是占 8 位。

---

`OptionsContext` 里面使用 `SpecifierOpt` 的时候，是以**数组**的方式使用的，后面会跟一个数组的长度，如下：

```
SpecifierOpt *codec_names; //指针就是数组
int        nb_codec_names; //数组长度
```

`codec_names` 是指编解码器的名称，所以 `ffmpeg.exe ` 可以指定多个解码器来解码文件，如下：

```
ffmpeg.exe -c:v h264_cuvid -c:v h264_mf -i juren.mp4 juren.flv
```

这样会优先使用 `h264_cuvid` 硬件解码器解码，如果 `ffmpeg.exe` 没有编译硬件解码器，就会使用 `h264_mf`

---

下面讲解一下 `SpecifierOpt` 的解析代码，如下：

![1-8](open_files\1-8.png)

变量 so 直接指向  `OptionsContext` 里面 `SpecifierOpt` 类型的某个字段了。

下面的代码，实际是为了提取出 `a` 或者 `v` 字符， `str` 等于 a 或者 v，代表这是音频还是视频的选项。

```
char *p = strchr(opt, ':');
char *str;
...省略..
str = av_strdup(p ? p + 1 : "");
```

例如 `-c:v` ，`-c:a`，可以指定音频或者视频的编码器。上面的代码这样操作之后，就可以把 str 指向 a 或者 v。

---

变量 so 一开始其实是一个 NULL 指针，需要用 `grow_array()` 函数来动态扩容。

最后把 `dst` 指向 `SpecifierOpt` 里面的 u 联合体字段，这个其实比较通用，

`SpecifierOpt` 的 u 跟，`OptionDef` 的 u 其实是有点类似，**可以通用后面的赋值逻辑**。

```
typedef struct SpecifierOpt {
    ...省略...
    union {
        uint8_t *str;
        int        i;
        int64_t  i64;
        uint64_t ui64;
        float      f;
        double   dbl;
    } u;
} SpecifierOpt;
```

```
typedef struct OptionDef {
     ...省略...
     union {
        void *dst_ptr;
        int (*func_arg)(void *, const char *, const char *);
        size_t off;
    } u;
    ...省略...
} OptionDef;
```

---

至此，`parse_option()` 把 `OptionGroup` 转成 `OptionsContext` 的逻辑分析完毕了。

得到 `OptionsContext` 之后，就会调指针函数 `open_file()` 去打开输入或者输出文件。

`open_file` 是一个指针函数，可能指向 [open_input_file()](https://ffmpeg.xianwaizhiyin.net/ffmpeg/open_input_file.html) ，也可能指向 [open_output_file()](https://ffmpeg.xianwaizhiyin.net/ffmpeg/open_output_file.html)。
