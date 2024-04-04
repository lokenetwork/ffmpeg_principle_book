# FFmpeg的视频format滤镜介绍—FFmpeg API教程

<div id="meta-description---">视频的 format 滤镜是一个非常常用的滤镜，用来转换图像的格式，例如可以把 AV_PIX_FMT_YUV420P 转成 AV_PIX_FMT_RGB24。</div>

视频的 `format` 滤镜是一个非常**常用**的滤镜，用来转换图像的格式，例如可以把 `AV_PIX_FMT_YUV420P` 转成 `AV_PIX_FMT_RGB24`。

我们可以用以下命令查询 format 滤镜支持的参数：

```
ffmpeg -hide_banner 1 -h filter=format
```

![1-1](format_filter\1-1.png)

从上图可以看到，只有一个参数 `pix_fmts`，但是这个 `pix_fmts` 这个参数的 `value` 是一个**列表**。列表里面可以只有一种图像格式，也可以有多种图像格式。如果列表里面是多种，转换的时候，会转成其中一种图像格式。

为什么会是列表呢？

我估计是因为某些图像格式，之间是不能进行转换，所以传一个列表，能转成哪种就选择哪种来转换。

这个列表**通常是**编码器支持的图像格式的列表，大部分编码器都支持两种以上的图像格式作为输入。 `format` 滤镜转换好格式之后，就可以丢给 编码器进行编码了。

------

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/format_filter)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

重点代码如下：

![1-2](format_filter\1-2.png)

![1-3](format_filter\1-3.png)

运行结果如下：

![1-4](format_filter\1-4.png)

可以看到，原来的像素格式是 0（AV_PIX_FMT_YUV420P），经过 `format` 滤镜处理之后，就变成 1 （AV_PIX_FMT_YUYV422）。

------

由于初始化滤镜的时候，`format` 参数传的是像素格式的**字符串**，如何找到这个字符串呢？

例如现在你想把图像格式转成 `AV_PIX_FMT_NV12`，这是一个枚举数字，这个 `AV_PIX_FMT_NV12` 对应的字符串是什么呢？

答：可以从 `libavutil/pixdesc.c` 里面的 **av_pix_fmt_descriptors** 数组找到对应的字符串描述，如下：

![1-5](format_filter\1-5.png)

上图中的 `.name` 字段就是字符串，把 `nv12` 这个字符串替换掉  `[main]format=yuyv422[result]` 里面的 `yuyv422`，`format` 滤镜就会转成 `nv12` 的图像格式输出。

也可以用 `av_pix_fmt_desc_get()` 函数来获取 `AV_PIX_FMT_NV12` 对应的字符串。

------

细心的读者可能注意到了，一开始讲的 `pix_fmts` 在代码里面没有用到，其实上面代码演示的是 `ffmpeg` 滤镜API的**简写用法**，写全是下面这样的。

```
[main]format=yuyv422[result];
等价于
[main]format=pix_fmts=yuyv422[result];
```

读者可以把这两种写法都测试一下，结果其实是一样的。

简写的时候，你可以不写 `key`，只写 `value`，但是 `value` 必须按默认的顺序写上去。

滤镜字符串，如果有多个 `key=value` ，之间是用冒号 `:` 隔开的，不过 `format` 滤镜只有一个 `key`，就是 `pix_fmts`。

------

由于 `pix_fmts` 可以指定多种图像格式，而编码器编码的时候必须要知道输入的是什么图像格式，所以有一个函数可以获取滤镜容器最后输出的图像格式。

那就是 `av_buffersink_get_format()`，当 `avfilter_graph_parse2()` 执行完之后，滤镜容器输出的图像格式就是确定的了，你可以用  `av_buffersink_get_format()` 函数来获取最后输出的图像格式。

这个在 `ffmpeg.c` 里面也是这样做的，如下：

![1-6](format_filter\1-6.png)

------

转换图像格式也可以使用 `sws_scale()` 的函数，`sws` 的全称是 **software scale**。

`sws_scale()` 函数不仅仅可以转换图像格式，还可以转换宽高 等等。推荐阅读《[sws_scale图像缩放函数介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/sws_scale.html)》

------

至此，`format` 滤镜介绍完毕，`format` 是视频的滤镜，自然音频也会有一个类似的滤镜，那就是 `aformat`，请继续阅读《[FFmpeg的音频aformat滤镜介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/aformat_filter.html)》

