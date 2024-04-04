# FFmpeg的split滤镜介绍—FFmpeg API教程

<div id="meta-description---">本文介绍 split 滤镜的用法以及 avfilter_link 函数 的具体用法。</div>

本文介绍 split 滤镜的用法以及 `avfilter_link` 函数 的具体用法。

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/split)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

------

上下文 这个词读起来不太通顺，`context` 这个单词我后面把它翻译成**实例**吧，滤镜实例。

------

之前在 **scale-1** 项目留了一个问题，就是 `avfilter_link` 函数 的第二第四个参数的作用是什么。 `avfilter_link` 函数 的定义如下：

```
int avfilter_link(AVFilterContext *src, unsigned srcpad,
                  AVFilterContext *dst, unsigned dstpad)
```

为什么会有这两个参数 `srcpad` 跟 `dstpad`，这是因为 FFmpeg 的滤镜是有很多种类型，各种各样。

例如之前的 scale 滤镜，只接受1个输入流，然后输出1个流。

但是 split 滤镜可以接受1个输入流，然后输出2个或者多个流。

而 overlay 滤镜可以接受2个输入流，但是只输出1个流。

因此  `srcpad` 参数是用来定位 滤镜实例 里面的第几个输出流，而  `dstpad` 参数是用来定位 滤镜实例 里面的第几个输入流。

------

下面就用 **split-1** 项目里面的代码来理解 `srcpad` 跟 `dstpad` 参数的作用。

![split-1-2](split\split-1-0.png)

split 滤镜的参数是 `outputs=2` ，代表分出 两个流，你也可以把这个数字改大，分出多个流。

滤镜实例 创建之后，里面的 `nb_inputs` 跟 `nb_ouputs` 字段 就表明了这个 滤镜实例 有多少个输入，有多少个输出，下图是 split 滤镜实例的截图：

![split-1-1](split\split-1-1.png)



------

接着讲解一下如何连接 `split` 滤镜实例 里面的输入跟输出。代码如下：

![split-1-2](split\split-1-2.png)

由于 `split` 分出来两个流，只能用 接受两个流 `overlay` 滤镜来接收，中间使用了 `scale` 滤镜来进行缩放。

 `overlay` 滤镜的规则是把第二个输入流覆盖到第一个输入流上面。可以通过 参数设置覆盖的位置。因此第二个流最好比第一个流画面要小。

因此 **split-1** 项目里面滤镜实例之间的连接过程图如下：

![split-1-3](split\split-1-3.png)

------

从 **split-1** 项目的代码可以看出，这种一个一个滤镜实例创建，然后 `avfilter_link` 链接起来的操作非常繁琐。

下面演示一下，之前那种最简单的滤镜函数用法，就是 写好字符串的滤镜连接规则，然后丢进去  `avfilter_graph_parse2` 函数。

代码在 **split-simple** 项目，代码如下：

```
av_bprintf(&args,
        "buffer=video_size=%dx%d:pix_fmt=%d:time_base=%d/%d:pixel_aspect=%d/%d:frame_rate=%d/%d[main];"
        "[main]split[v0][v1];"
        "[v0]scale=%d:%d[v2];"
        "[v1][v2]overlay=%d:%d[result];"
        "[result]buffersink",
        frame->width, frame->height, frame->format, tb.num,tb.den,sar.num, sar.den,fr.num, fr.den,
        frame->width/4,  frame->height/4,
        frame->width/4*3,  frame->height/4*3);
```

![split-1-4](split\split-1-4.png)

------

上面的代码，`/4*3` 为了把缩略图覆盖在右下角，生成的图片如下：

![split-1-5](split\split-1-5.png)

因此，用字符串来配置滤镜规则，来调滤镜函数是最简单的，没有之一。这样根本不需要用到 `AVFilterInOut` 或者  `avfilter_link` 函数连来连去。





