# FFmpeg滤镜API教程

<div id="meta-description---">FFmpeg滤镜API教程</div>

FFmpeg 的 libavfilter 滤镜子系统当初是 **Michael Niedermayer** 引入的。据说 FFmpeg 的 Filter 是借鉴的 mplayer，而 mplayer 的 FIlter 又跟 Windows Media 的 Filter 有异曲同工之妙。

FFmpeg 采用了 Filter Graph 的模型来管理整个数据流的处理，参与数据处理的各个功能模块叫做 Filter（滤镜）。

普通的用户在使用音视频剪辑软件的时候，会把滤镜这个词理解为 app 上提供的一些特效，例如 变场，镜像，加水印 等等。

但是在 FFmpeg 音视频开发领域，Filter（滤镜）是指对音视频数据的处理，包括 裁剪，转换采样率格式，转换封装格式 等等，这些很简单的功能，在 FFmpeg 里面也是 一个 Filter。

所以，你可以把 Filter（滤镜）看成是一个大杂烩，有很多功能，这些功能都是用来对音频或者视频的数据进行处理。

---

各个 Filter 会在 Filter Graph 中 按照一定的顺序**连接**起来，流水线式 地 协同工作，如下：

![1-1](filter_api\1-1.jpg)

Filter 大致可以分为 3 类，如下：

**1，buffer Filter**，输入源 `Filter`，负责接受 `AVFrame` 的输入。可以调 `av_buffersrc_add_frame_flags()` 函数 往 这个 `FIlter` 输入 `AVFrame`。

在复杂滤镜场景下，输入源  `FIlter` 可以有多个，同时接受多个输入流的数据输入。

**2，功能性 Filter**，功能性 `Filter` 负责处理音视频数据。例如 `scale Filter` 就是一个功能性的 `Filter`，还有  `overlay`，`colorkey`，`drawtext` 等等，这些都属于功能性 `Filter`。

**3，buffersink Filter**，输出 `Filter`，可以调 `av_buffersink_get_frame_flags()` 函数，从这个 `buffersink FIlter` 里面读取出来已经处理好的 `AVFrame`。

在复杂滤镜场景下，输出  `FIlter` 也是可以有多个的，同时输出多个流。

---

**跟滤镜有关的数据结构有以下：**

1. `AVFilterGraph`，**滤镜容器**，里面可以有多个 **滤镜上下文**
2. `AVFilterInOut`，滤镜链表，`avfilter_graph_parse2` 函数有时候会设置这个结构体，开放输入跟输出给**其他的滤镜上下文**来链接。
3. `AVFilterContext`，**滤镜上下文**，可以看成是滤镜的实例
4. `AVFilter`，滤镜信息。

**跟滤镜有关的API函数有以下：**

1. `avfilter_graph_alloc`，创建**滤镜容器**。
2. `avfilter_get_by_name`，根据字符串名字找出 `AVFilter`
3. `avfilter_graph_create_filter` ，根据 `AVFilter` 来创建滤镜上下文。同时会把新创建的滤镜上下文放进去滤镜容器。
4. `avfilter_link`，连接两个**滤镜上下文**。
5. `avfilter_graph_parse2`，根据 传递的字符串语法 来创建 一个 或者 多个 滤镜上下文，多个滤镜会根据语法自动连接。同时会把新创建的滤镜上下文放进去滤镜容器。
6. `avfilter_graph_config`，正式打开滤镜容器。
7. `avfilter_graph_get_filter`，根据名称获取 滤镜容器内部的某个 **滤镜上下文**。
8. `av_buffersrc_add_frame_flags`，往 滤镜上下文 发送一个 `AVFrame`，让滤镜进行处理。
9. `av_buffersink_get_frame_flags`，从滤镜上下文读取已经处理好的 `AVFrame`。

---

参考资料：

1，《FFmpeg从入门到精通》第12页 - 刘歧

2，《Window Media编程导向》第12章 - 陆其明





