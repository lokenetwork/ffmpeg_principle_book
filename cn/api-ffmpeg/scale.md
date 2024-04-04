# FFmpeg的scale滤镜介绍—FFmpeg API教程

<div id="meta-description---">本文介绍FFmpeg 滤镜函数的三种用法，以scale滤镜来介绍这三种用法</div>

FFmpeg 的滤镜 API 其实有 3 种调用方法，我个人觉得他是 3 种用法，如下：

**1，**用 `avfilter_graph_create_filter` 一个一个地创建滤镜（`AVFilterContext`），然后用 `avfilter_link` 函数把各个滤镜的输入输出连接起来，这种方式比较灵活，但是非常繁琐。

**2，**下面的命令定义了一个滤镜字符串 `"[0:v]scale=iw/2:ih/2"`  ，直接使用 `avfilter_graph_parse2` 来解析这个字符串。`avfilter_graph_parse2` 函数内部会根据字符串的语法规则把所有滤镜链接起来。

```
ffmpeg.exe -i juren-30s.mp4 -filter_complex "[0:v]scale=iw/2:ih/2" output.mp4 -y
```

上面的 `[0:v]` ，代表取第一个输入文件的视频流。上面这种就是 `ffmpeg.c` 里面的滤镜用法。实际上`[0:v]` 的用法也是比较繁琐，这样会开放输入跟输出给其他滤镜实例来连接，也就是说用了  `[0:v]`，那你必须 调 `avfilter_graph_create_filter` 新建 buffer/buffersink 滤镜实例来连接 `AVFilterInOut`，如果有多个输入文件，你要调多次 `avfilter_graph_create_filter` 。

------

**3，**第三种使用滤镜的方法，其实也是定义一个字符串，然后丢给 `avfilter_graph_parse2` 函数，只不过这个字符串是这样的。如下：

```
"buffer=video_size=%dx%d:pix_fmt=%d:time_base=%d/%d:pixel_aspect=%d/%d:frame_rate=%d/%d[main];"
"[main]scale=%d:%d[result];"
"[result]buffersink"
```

上面的字符串直接定义了  `buffer/buffersink`  的规则，上面的字符串丢给 `avfilter_graph_parse2` 的时候，他会自动创建 `buffer/buffersink`  滤镜。如果有多个输入文件，直接加一行 `buffer` 字符串即可。

第三种用法，我个人觉得是最简单的，根本不用管那个 `AVFilterInOut` 的数据结构。

---

本文就以 scale滤镜 为例，来演示上面这 3 种使用滤镜的方法，`scale` 是一个可以调整图像宽高的滤镜。其他的功能性滤镜也是一样的用法。

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/scale)，里面有 3个项目。


------
现在先来讲第三种用法，因为最简单，请打开 **scale-3**  项目的代码。

下面就结合本文的 **scale-3** 项目代码来理解一下上面的数据结构跟函数，请看下图：

![scale-1-1](scale\scale-1-1.png)

上图是初始化 跟打开 一个滤镜 的代码，函数调用如下：

`avfilter_graph_alloc` ➔ `avfilter_graph_parse2`  ➔ `avfilter_graph_config`。

上面这 3个函数的调用比较容易看懂。比较不容易明了的是下面这两行代码：

```
 //根据 名字 找到 AVFilterContext
 mainsrc_ctx = avfilter_graph_get_filter(filter_graph, "Parsed_buffer_0");
 resultsink_ctx = avfilter_graph_get_filter(filter_graph, "Parsed_buffersink_2");                    
```

`avfilter_graph_parse2`  函数接受字符串 "Parsed_buffer_0" ，"Parsed_buffersink_2"。

这两个字符串是怎么算出的呢？

往 滤镜容器 创建 滤镜上下文 的时候，每个 滤镜上下文 都有一个名字的，这个名字的规则如下：

![scale-1-2](scale\scale-1-2.png)

上图代码中的 `name` 就是 字符串里面的 `buffer` 或者 `buffersink`。 index 是什么呢？ index 代表第几个滤镜上下文。

**第一个滤镜上下文**是 `buffer= .... [main]` ，index 是 0，拼起来就是 `Parsed_buffer_0`。这是一个 buffer 滤镜（输入滤镜）

**第二个滤镜上下文**是 `[main]scale=%d:%d[result]`，index 是 1，这是一个 `scale` 滤镜，拼起来是 `Parsed_scale_1` ，不过这个滤镜我们 scale-3 项目代码没调出来不用管。

**第三个滤镜上下文**是 `[result]buffersink`，index 是 2，这是一个 `buffersink` 滤镜（输出滤镜），拼起来是 `Parsed_buffersink_2` 

------

现在，只要往 `buffer` 滤镜上下文 发 `AVFrame`，从 `buffersink` 滤镜上下文 读 `AVFrame`，就可以了。图像缩小了一倍。代码如下：

![scale-1-3](scale\scale-1-3.png)

------

最后，就需要释放 滤镜相关的内存，释放也很简单，因为无论是 通过 `avfilter_graph_create_filter` 还是 `avfilter_graph_parse2` 创建的滤镜上下文，都放进去了滤镜容器里面。

所以只需要调 `avfilter_graph_free` 来释放  滤镜容器 即可，容器里面的滤镜上下文都会全部释放。

```
//释放滤镜。
avfilter_graph_free(&filter_graph);
```

------

下面我们继续讲 第一种 最原始的创建滤镜的方法，里面没有用到 字符串滤镜语法。完全是一个一个滤镜创建，然后连接起来的。

请打开 **scale-1**  项目的代码。请看下图：

![scale-1-4](scale\scale-1-4.png)

上图中的函数调用流程如下：

`avfilter_graph_alloc` ➔ `avfilter_graph_create_filter`  ➔  `avfilter_link` ➔ `avfilter_graph_create_filter`  ➔  `avfilter_link` ➔ `avfilter_graph_config`

可以看出了，如果滤镜很多，要写很多代码，但是这种原始的方式是最灵活的，你想怎么链接都可以。

`avfilter_link` 函数有一个重点，就是第二第四个参数，这是一个下标，本文填0即可，这两个参数特别重要，在 [《FFmpeg的split滤镜介绍》](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/split.html)文章有讲解。

 **scale-1**  项目的代码 没有使用 `avfilter_graph_parse2` 函数。

至此，第一种使用滤镜的方法也讲解完毕了。




------

 最后讲 第二种 调用 滤镜函数的方法，代码在  **scale-2**  项目里面，这种方法就是 `ffmpeg.c` 里面的滤镜用法，可以说是官方用法。

 **scale-2**  项目的代码虽然跟  `ffmpeg.c` 用的同一种滤镜调用方法，但是 `ffmpeg.c` 里面的滤镜功能更加复杂，因为他要根据 `[0:v]` 定位到哪个文件哪个流。

FFmpeg 为了实现这些命令行滤镜功能，导致 `ffmpeg.c` 里面的滤镜代码非常复杂，但实际上，滤镜函数的调用，难度其实是跟  **scale-2**  项目一样的。

下面就我们一起探索  **scale-2**  项目的代码，请看下图：

![scale-1-5](scale\scale-1-5.png)

上面的代码 是 创建 `buffer ctx` 跟 `buffersink ctx`，跟之前一样。

------

继续看后面代码，如下：

![scale-1-6](scale\scale-1-6.png)

实际上 第二种方法，我个人觉得是专门 为 `ffmpeg.c` 准备的，如果你是一个库使用者，使用本文的第三种方法即可。

**scale-2** 项目由于传给 `avfilter_graph_parse2` 函数的字符串没有定义 `buffer` 跟 `buffersink`，所以默认不会创建这两个 滤镜上下文。但是他会提供 `AVFilterInOut` 开放入口跟出口给你自己创建的  `buffer` 跟 `buffersink` 来连接。

上面我打印了一些 cur 变量的值，运行结果如下：

![scale-1-7](scale\scale-1-7.png)



**scale-2**  项目的代码看起来也比较容易理解，这是因为我没有管那个 `[0:v]`，我的输入文件是写死的。一旦要实现 ffmpeg 命令行滤镜的功能，代码就会异常复杂。

**这个 `[0:v]` 标记语法是可以自定义的，定义成怎样的都可以，里面的 0 跟 v 代表什么完全是由你自己的代码决定。**

你可以用 `[a:vvv]` 来表示第一个文件的视频流，这也是可以的，**这只是一个标示**，这个 `[a:vvv]` 标示会被解析成 `AVFilterInOut` 结构，然后你就可以创建一个 `buffer` 入口滤镜来链接这个 `AVFilterInOut` 。

具体 `ffmpeg.exe` 转换器 配置滤镜，解析 `[0:v]` 的逻辑，推荐后面再阅读《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》



