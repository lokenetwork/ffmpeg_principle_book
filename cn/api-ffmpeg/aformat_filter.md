# FFmpeg的音频aformat滤镜介绍—FFmpeg API教程

<div id="meta-description---">前面介绍了 FFmpeg 的 format 视频格式滤镜，那很显然，音频也会有一个格式滤镜，用来转换音频采样格式，调整采样率或者声道布局。音频的格式滤镜叫 aformat，前面加了个 a 而已。</div>

前面介绍了 FFmpeg 的 `format` 视频格式滤镜，那很显然，音频也会有一个格式滤镜，用来**转换音频采样格式**，**调整采样率或者声道布局**。

音频的格式滤镜叫 `aformat`，前面加了个 `a` 而已。

这是 FFmpeg 整个开源项目的命名习惯，不仅仅是格式滤镜，还有 `buffer` 滤镜 与 `abuffer` 滤镜，这两个分别是视频，音频的入口滤镜。而出口滤镜是 `buffersink` 与 `abuffersink`。

总之，如果你遇到一个视频模块 叫 `xxx`，通常你在前面加个 `a`变成 `axxx`， 就是音频的模块了。

------

我们可以用以下命令查询 `aformat` 滤镜支持的参数：

```
ffmpeg -hide_banner 1 -h filter=aformat
```

![1-1](aformat_filter\1-1.png)

可以看到，`aformat` 滤镜支持 3 个参数，`sample_fmts`（采样格式），`sample_rates`（采样率），`channel_layouts`（声道布局）。

这 3 个参数也是列表的形式，跟 `format` 视频格式滤镜一样。

---

`aformat` 音频格式滤镜的示例代码在 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/aformat_filter) 可以下载，重点代码如下：

![1-2](aformat_filter\1-2.png)

![1-3](aformat_filter\1-3.png)

整个项目的运行结果如下：

![1-4](aformat_filter\1-4.png)

可以看到，`juren-30s.mp4` 的音频帧，原本是 `fltp` 格式的，经过 `aformat` 滤镜转换之后，就变成了 `s64` 格式的了，同时采样率跟声道布局也进行了调整。

------

FFmpeg 里面定义的音频采样格式，一共有 12 种，枚举 `AV_SAMPLE_FMT_NB` 的值就是 12。全部都定义在 `llibavutil/samplefmt.h` 里面，如下：

```
enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,
    AV_SAMPLE_FMT_U8,          ///< unsigned 8 bits
    AV_SAMPLE_FMT_S16,         ///< signed 16 bits
    AV_SAMPLE_FMT_S32,         ///< signed 32 bits
    AV_SAMPLE_FMT_FLT,         ///< float
    AV_SAMPLE_FMT_DBL,         ///< double

    AV_SAMPLE_FMT_U8P,         ///< unsigned 8 bits, planar
    AV_SAMPLE_FMT_S16P,        ///< signed 16 bits, planar
    AV_SAMPLE_FMT_S32P,        ///< signed 32 bits, planar
    AV_SAMPLE_FMT_FLTP,        ///< float, planar
    AV_SAMPLE_FMT_DBLP,        ///< double, planar
    AV_SAMPLE_FMT_S64,         ///< signed 64 bits
    AV_SAMPLE_FMT_S64P,        ///< signed 64 bits, planar

    AV_SAMPLE_FMT_NB           ///< Number of sample formats. DO NOT USE if linking dynamically
};
```

上面这些值是数字，因为滤镜里面使用的是字符串，所以这些枚举数字对应的字符串在 `llibavutil/samplefmt.c` 里面，如下：

```
/** this table gives more information about formats */
static const SampleFmtInfo sample_fmt_info[AV_SAMPLE_FMT_NB] = {
    [AV_SAMPLE_FMT_U8]   = { .name =   "u8", .bits =  8, .planar = 0, .altform = AV_SAMPLE_FMT_U8P  },
    [AV_SAMPLE_FMT_S16]  = { .name =  "s16", .bits = 16, .planar = 0, .altform = AV_SAMPLE_FMT_S16P },
    [AV_SAMPLE_FMT_S32]  = { .name =  "s32", .bits = 32, .planar = 0, .altform = AV_SAMPLE_FMT_S32P },
    [AV_SAMPLE_FMT_S64]  = { .name =  "s64", .bits = 64, .planar = 0, .altform = AV_SAMPLE_FMT_S64P },
    [AV_SAMPLE_FMT_FLT]  = { .name =  "flt", .bits = 32, .planar = 0, .altform = AV_SAMPLE_FMT_FLTP },
    [AV_SAMPLE_FMT_DBL]  = { .name =  "dbl", .bits = 64, .planar = 0, .altform = AV_SAMPLE_FMT_DBLP },
    [AV_SAMPLE_FMT_U8P]  = { .name =  "u8p", .bits =  8, .planar = 1, .altform = AV_SAMPLE_FMT_U8   },
    [AV_SAMPLE_FMT_S16P] = { .name = "s16p", .bits = 16, .planar = 1, .altform = AV_SAMPLE_FMT_S16  },
    [AV_SAMPLE_FMT_S32P] = { .name = "s32p", .bits = 32, .planar = 1, .altform = AV_SAMPLE_FMT_S32  },
    [AV_SAMPLE_FMT_S64P] = { .name = "s64p", .bits = 64, .planar = 1, .altform = AV_SAMPLE_FMT_S64  },
    [AV_SAMPLE_FMT_FLTP] = { .name = "fltp", .bits = 32, .planar = 1, .altform = AV_SAMPLE_FMT_FLT  },
    [AV_SAMPLE_FMT_DBLP] = { .name = "dblp", .bits = 64, .planar = 1, .altform = AV_SAMPLE_FMT_DBL  },
};
```

**你也可以通过 `av_get_sample_fmt_name()` 函数来获取数字对应的字符串。**

------

转换音频格式也可以使用 `swr_convert()` 的函数，`swr` 的全称是 **software resample**。

不过我个人觉得 `swr_convert()` 重采样函数使用起来有点复杂。不像滤镜的语法那么统一，只需往入口滤镜丢数据，然后往出口滤镜读数据就行了。

`swr_convert()` 相关介绍推荐阅读《[swr_convert音频重采样介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/swr_convert.html)》

------

至此，`aformat` 音频格式滤镜介绍完毕，
