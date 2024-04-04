# FFplay视频滤镜分析—ffplay.c源码分析

<div id="meta-description---">FFplay播放器的命令行是可以指定多个视频滤镜，然后按 w 键切换查看效果的，命令如下</div>

**FFplay** 播放器的命令行是可以指定多个视频滤镜，然后按 `w` 键切换查看效果的，命令如下：

```
# linux 命令
ffplay.exe -x 400 \
-vf "drawtext=fontsize=200:fontfile=FreeSerif.ttf:text='FFmpeg':x=150:y=100" \
-vf "drawtext=fontsize=200:fontfile=FreeSerif.ttf:text='Principle':x=150:y=100" \
-i juren.mp4
```

```
# windows 命令
ffplay.exe -x 400 ^
-vf "drawtext=fontsize=200:fontfile=FreeSerif.ttf:text='FFmpeg':x=150:y=100" ^
-vf "drawtext=fontsize=200:fontfile=FreeSerif.ttf:text='Principle':x=150:y=100" ^
-i juren.mp4
```

扩展知识：Linux 里面连接两行命令用的是 `\` 斜杠符号，而 Windows 系统连接两行命令用的是 `^` 平方符号。

`FreeSerif.ttf` 是字体文件，如果没有这个字体文件，请网上下载保存到跟 `ffplay.exe` 同样的目录

上面的命令定义了两个视频滤镜，这两个滤镜都是往视频加一个文字水印，那是不是两个滤镜都会生效呢？

不是，只会默认第一个滤镜生效，但是你可以按 `w` 键切换到第二个滤镜，效果如下：

![1-1](video_filter\1-1.png)

![1-2](video_filter\1-2.png)

------

下面来讲一下 `ffpaly.c` 里面是怎么解析 `-vf` 参数的，如下：

![1-3](video_filter\1-3.png)

可以看到，是调的 `opt_add_vfilter()` 来处理 `-vf` 命令行参数的。

`opt_add_vfilter()` 函数的代码如下：

```
static const char **vfilters_list = NULL;
static int opt_add_vfilter(void *optctx, const char *opt, const char *arg)
{
    GROW_ARRAY(vfilters_list, nb_vfilters);
    vfilters_list[nb_vfilters - 1] = arg;
    return 0;
}
```

`vfilters_list` 是一个二级指针，`GROW_ARRAY()` 是 `ffmpeg` 封装的函数，可以动态扩容。

------

我们再来看一下处理 `w` 键事件的逻辑，如下：

![1-4](video_filter\1-4.png)

`w` 键的逻辑是，切换到下一个视频滤镜，如果没有下一个视频滤镜了，就切换到音频波形图。

当 `cur_stream->vfilter_idx` 产生变化的时候，就会触发重新配置视频滤镜的逻辑，如下：

![1-5](video_filter\1-5.png)

上图的  `is->vfilter_idx` 其实就是 `cur_stream->vfilter_idx`，`ffplay` 有些函数会把 `is` 这个名称换成 `cur_stream`，实际上他们是同一个指针。

------

`configure_video_filters()` 函数比较简单，所以只讲一个重点，就是它里面有一个 `ffmpeg` 像素格式到 `SDL` 像素格式的映射过程，这个如果你的项目也用到，可以抄过去这个表。

![1-6](video_filter\1-6.png)

```
static const struct TextureFormatEntry {
    enum AVPixelFormat format;
    int texture_fmt;
} sdl_texture_format_map[] = {
    { AV_PIX_FMT_RGB8,           SDL_PIXELFORMAT_RGB332 },
    { AV_PIX_FMT_RGB444,         SDL_PIXELFORMAT_RGB444 },
    { AV_PIX_FMT_RGB555,         SDL_PIXELFORMAT_RGB555 },
    { AV_PIX_FMT_BGR555,         SDL_PIXELFORMAT_BGR555 },
    { AV_PIX_FMT_RGB565,         SDL_PIXELFORMAT_RGB565 },
    { AV_PIX_FMT_BGR565,         SDL_PIXELFORMAT_BGR565 },
    { AV_PIX_FMT_RGB24,          SDL_PIXELFORMAT_RGB24 },
    { AV_PIX_FMT_BGR24,          SDL_PIXELFORMAT_BGR24 },
    { AV_PIX_FMT_0RGB32,         SDL_PIXELFORMAT_RGB888 },
    { AV_PIX_FMT_0BGR32,         SDL_PIXELFORMAT_BGR888 },
    { AV_PIX_FMT_NE(RGB0, 0BGR), SDL_PIXELFORMAT_RGBX8888 },
    { AV_PIX_FMT_NE(BGR0, 0RGB), SDL_PIXELFORMAT_BGRX8888 },
    { AV_PIX_FMT_RGB32,          SDL_PIXELFORMAT_ARGB8888 },
    { AV_PIX_FMT_RGB32_1,        SDL_PIXELFORMAT_RGBA8888 },
    { AV_PIX_FMT_BGR32,          SDL_PIXELFORMAT_ABGR8888 },
    { AV_PIX_FMT_BGR32_1,        SDL_PIXELFORMAT_BGRA8888 },
    { AV_PIX_FMT_YUV420P,        SDL_PIXELFORMAT_IYUV },
    { AV_PIX_FMT_YUYV422,        SDL_PIXELFORMAT_YUY2 },
    { AV_PIX_FMT_UYVY422,        SDL_PIXELFORMAT_UYVY },
    { AV_PIX_FMT_NONE,           SDL_PIXELFORMAT_UNKNOWN },
};
```

`configure_video_filters()` 函数最后会配置 `is->in_video_filter` 跟 `is->out_video_filter`，就是入口滤镜跟出口滤镜，只需要把解码出来的视频帧往 `in_video_filter` 丢，然后再从 `out_video_filter` 读就行了。

更多滤镜函数的用法，请阅读 《[FFmpeg滤镜API教程](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/filter_api.html)》 一章。





