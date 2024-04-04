# FFmpeg的overlay滤镜介绍—FFmpeg API教程

<div id="meta-description---">面介绍 FFmpeg 滤镜的文章，其实埋了一个坑，滤镜实例有输入跟输出。但是往 buffer 滤镜实例输入的 AVFrame 不是无限的，总会有读完文件的一刻。从 buffersink 滤镜实例 输出的 AVFrame 也不是无限的，总会有刷完的一刻。没有 AVFrame 可以输入了，怎么处理？没有 AVFrame 可以刷出来了，又怎么处理？这就是本文的重点，本文会通过 overlay 滤镜演示上面这些情况的代码如何写。</div>

前面介绍 FFmpeg 滤镜的文章，其实埋了一个坑，滤镜实例有输入跟输出。但是往 buffer 滤镜实例输入的 AVFrame 不是无限的，总会有读完文件的一刻。从 buffersink 滤镜实例 输出的 AVFrame 也不是无限的，总会有刷完的一刻。

没有 AVFrame 可以输入了，怎么处理？没有 AVFrame 可以刷出来了，又怎么处理？

这就是本文的重点，本文会通过 overlay 滤镜来写一个小 demo

---

本文用 overlay 滤镜实现一个往视频里面加 logo 水印图片的例子，代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/overlay-simple )

------

初始化跟创建 滤镜实例跟容器的代码如下：

![overlay-1-1](overlay\overlay-1-1.png)

这次我初始化的时候，在字符串里面写了 两个 buffer，然后我只需要根据 名字 找到 `AVFilterContext` 即可，非常方便。

------

然后由于 logo 视频流实际上是一个图片，所以这个 logo 流只有一帧图片，一旦没有更多的 AVFrame 输入，就需要用 `av_buffersrc_close` 函数关闭 buffer 滤镜实例，如下：

![overlay-1-2](overlay\overlay-1-2.png)

`av_buffersrc_close` 函数里面有一个 pts 参数，这是下一帧的pts，可以通过当前帧的 pts 跟他的 duration 计算出来。

我实验了一下，直接往 buffer 滤镜里面发一个 NULL 的 `AVFrame` 也是可以的，但是正规的做法是调 `av_buffersrc_close` 函数，ffmpeg.c 里面也是这么做的。

------

在所有的 buffer 输入滤镜实例都**关闭**之后，`av_buffersink_get_frame_flags` 函数就会返回 `AVERROR_EOF` 告诉你没有更多的 AVFrame 输出了，滤镜处理就完了。代码如下：

![overlay-1-3](overlay\overlay-1-3.png)

------

从输入输出 EOF 的处理来看，滤镜实例 跟 之前的 编码器，解码器 实例拥有相同的逻辑，他们都有输入输出，都需要对末尾（EOF）数据做处理。

------

代码运行结果如下：

![overlay-1-4](overlay\overlay-1-4.png)

在 qt 的生成目录，可以看到以下文件。

![overlay-1-5](overlay\overlay-1-5.png)

用 7yuv 软件查看可以发现，logo 已经顺利印上去了。

![overlay-1-6](overlay\overlay-1-6.png)

------

本项目的效果可以用以下命令实现：

```
ffmpeg -i juren-30s.mp4 -i logo.jpg -filter_complex "[0:v][1:v]overlay=x=10:y=10" output.mp4 -y
```

扩展知识：可以通过 下面的命令查询 具体滤镜的参数。

```
ffmpeg -h filter=overlay
```

![overlay-1-7](overlay\overlay-1-7.png)

------

如果想要透明的 logo，可以使用本文提供的 png 图片。不过本文的 ffmpeg 库没有把 png 的解码器编译进去，所以解码不了 png， 需要读者自己编译一次 FFmpeg 的库。

