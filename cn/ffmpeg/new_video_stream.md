# new_video_stream添加视频输出流—ffmpeg.c源码分析

<div id="meta-description---">new_video_stream添加视频输出流</div>

`new_video_stream()` 函数的流程相对来说比较简单，主要的逻辑如下：

**1，**调 `new_output_stream()` 函数来创建 `OutputStream` 输出流，以及 `AVCodecContext` 编码器上下文。

`new_output_stream()` 是一个公共函数，创建 音频流，数据流，字幕流都用了它。

`new_output_stream()` 会把命令行的一些公共参数赋值给 `OutputStream` 跟 `AVCodecContext`。

这些公共参数是指音频，视频，字幕都可能会有的参数。因为 `new_output_stream()` 是一个公共函数。

**2，**调 `MATCH_PER_STREAM_OPT()` 宏函数，把 `OptionsContext` 里面**视频相关的参数**，赋值给 给 `OutputStream` 跟 `AVCodecContext`。

流程图如下：

![1-1](new_video_stream\1-1.jpg)

---

可以看到，实际上就两步，`new_video_stream()` 肯定会创建视频的输出流，还有视频的编码器实例。

公共参数，就在  `new_output_stream()` 函数 里面赋值了。

视频相关的参数，就在 `new_video_stream()` 函数再赋值。

`new_video_stream()` 跟  `new_output_stream()` 函数都调用了多次 `MATCH_PER_STREAM_OPT()` 宏函数来提取 `OptionsContext` 的内容，

 `MATCH_PER_STREAM_OPT()` 其实是 `MATCH_PER_TYPE_OPT()` 的**兄弟函数**，

```
#define MATCH_PER_TYPE_OPT(name, type, outvar, fmtctx, mediatype)\
{\
    int i;\
    for (i = 0; i < o->nb_ ## name; i++) {\
        char *spec = o->name[i].specifier;\
        if (!strcmp(spec, mediatype))\
            outvar = o->name[i].u.type;\
    }\
}
```

```
#define MATCH_PER_STREAM_OPT(name, type, outvar, fmtctx, st)\
{\
    int i, ret, matches = 0;\
    SpecifierOpt *so;\
    for (i = 0; i < o->nb_ ## name; i++) {\
        char *spec = o->name[i].specifier;\
        if ((ret = check_stream_specifier(fmtctx, st, spec)) > 0) {\
            outvar = o->name[i].u.type;\
            so = &o->name[i];\
            matches++;\
        } else if (ret < 0)\
            exit_program(1);\
    }\
    if (matches > 1)\
       WARN_MULTIPLE_OPT_USAGE(name, type, so, st);\
}
```

这两个函数，只有最后一个参数，第五个参数是不一样的。

`mediatype` 通常是 `a` 或者 `v`，也就是根据 `a` 还是 `v` 字符来提取 `OptionsContext` 里面音频或者视频的选项。

`st` 是 `AVStream`，所以如果 `AVStream` 是音频，就提取 `OptionsContext` 里面的音频选项，如果是视频就提取视频。

这两个函数的宏实现看起来有点复杂，但他们的区别就是这么一点区别。

---

至此，`new_video_stream()` 函数的源码分析完毕。`new_audio_stream()` 跟 `new_video_stream()` 类似，里面都调了 `new_output_stream()` 。

`new_audio_stream()` 主要是提取`OptionsContext` 里面音频选项，对  `OutputStream` 输出流，以及 `AVCodecContext` 编码器 进行赋值操作。

---

补充一点：虽然 `new_video_stream()` 里创建了 编码器实例，但是还没真正打开编码器的。打开编码器，需要等到解码出第一帧 `AVFrame`。才会打开编码器。

为什么是这样呢？

因为 f`fmpeg.exe` 的逻辑，是只有在解码出第一帧 `AVFrame` 的时候，才去用 `avfilter_graph_config()` 打开 `FilterGragh` ，这样才能从**出口滤镜**读取到 输出的宽高是多少。

`ffmpeg.exe` 比较谨慎，他可能不太相信容器层记录的宽度，也有可能有些容器根本没记录宽高，所以他必须等到解码出 `AVFrame`，才能确定输入的宽高，确定了输入的宽高，才能创建 `buffer`入口滤镜，创建了入口滤镜，才能打开 `FilterGragh` 。

> TODO：这个逻辑非常重要，在本章结尾的时候再重复讲一次。

最后是在 `init_output_stream_encode()` 里面，从滤镜出口里面获取的宽高，如下：

```
enc_ctx->width  = av_buffersink_get_w(ost->filter->filter);
enc_ctx->height = av_buffersink_get_h(ost->filter->filter);
enc_ctx->sample_aspect_ratio = ost->st->sample_aspect_ratio
```

---

最后，推荐一下 clion 的 Call Hierarchy 功能，可以看到函数的调用流程，如下：

![1-2](new_video_stream\1-2.png)

大部分的 集成开发环境都有这个功能，你只需用 “工具名称” +  Call Hierarchy 关键词，即可搜索到相关教程。
