# init_complex_filtergraph初始化复杂滤镜—ffmpeg.c源码分析

<div id="meta-description---">init_complex_filtergraph初始化复杂滤镜，绑定 InputFilter，OutputFilter，InputStream，OutputStream</div>

`ffmpeg.exe` 的滤镜分为两种，简单滤镜 和 复杂滤镜。简单滤镜已经在 **FFmpeg转换器分析-基础篇** 一章的《[init_simple_filtergraph初始化简单滤镜](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph.html)》《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》文章中讲解过了。

现在来重温一下简单滤镜的知识，简单滤镜是指命令行没有使用 `-filter_complex` 选项，如下：

```
ffmpeg -i juren.mp4 juren.flv
```

```
ffmpeg -i juren.mp4 -vf "null" juren.flv
```

上面两条命令是等价的，即便你命令行没有使用 `-vf` 指定视频滤镜，在 `ffmpeg.c` 内部也会创建一个 `null` 的滤镜把输入输出连接起来。

而复杂滤镜的命令行示例如下：

```
ffmpeg -i juren-30s.mp4 -i logo.jpg -filter_complex "[1:v]scale=176:144[logo];[0:v][logo]overlay=x=0:y=0" output.mp4 -y
```

效果图如下：

![1-1](init_complex_filtergraph\1-1.png)

----

本文主要讲解 `-filter_complex` 在 `ffmpeg.c` 里面的实现。从下面的定义可以看出解析 `filter_complex` 参数会调用 `opt_filter_complex()` 函数。

![0-0](init_complex_filtergraph\0-0.png)

`opt_filter_complex()` 函数的定义如下：

![0-0-2](init_complex_filtergraph\0-0-2.png)

`opt_filter_complex()` 函数的代码量比较少，首先用 `GROW_ARRAY()` 宏函数动态扩容一下 `filtergraphs` 数组，**`GROW_ARRAY()` 宏函数可以把数组的容量加1**

然后用 `av_mallocz()` 一个 `struct FilterGraph` 结构体的内存，放进去 `filtergraphs` 数组。而`-filter_complex` 后面的参数字符串 `"[1:v]scale=176:144[logo]..."` 会被存储在 `graph_desc` 字段。

这里有一个重点就是  `graph_desc` 字段，这个字段只有是复杂滤镜才会有值，如果是简单滤镜，这个字段是 NULL。

那简单滤镜 `-vf` 后面的字符串存储在哪里的呢？

```
ffmpeg -i juren.mp4 -vf "split[main][tmp];[tmp]crop=iw:ih/2:0:0,vflip[flip];[main][flip]overlay=0:H/2" out.mp4
```

答：绑定在 `OutputStream` 的 `filters` 字段里。具体请阅读《[FFmpeg命令参数分析-vf](https://ffmpeg.xianwaizhiyin.net/ffmpeg/cmd_arg_vf.html)》

----

其实 简单滤镜 与 复杂滤镜，是非常相似的，因为他们共用一套数据结构，`struct FilterGraph`，`InputFilter`，`OutputFilter`，`InputStream`，`OutputStream`。

简单滤镜场景下，只有一个 `InputFilter`（输入），一个 `OutputFilter`（输出）。

复杂滤镜场景下，可以有一个，也可以有多个  `InputFilter`（输入），多个 `OutputFilter`（输出）。

`InputFilter` 最后是跟 `InputStream` 绑定在一起，而 `OutputFilter` 跟 `OutputStream` 绑定。

---

简单滤镜 与 复杂滤镜都是调 `configure_filtergraph()` 来正式打开滤镜容器的，但是他们的初始化函数不一样，以及初始化的位置也不一样。

简单滤镜 的初始化函数是 `init_simple_filtergraph()`，复杂滤镜的初始化函数是 `init_complex_filtergraph()`，他们在函数调用中的位置如下：

![0-1](init_complex_filtergraph\0-1.jpg)

从上面流程图 可以看出 ，`init_complex_filters()` 是在 `init_simple_filtergraph()` 之前执行的。

整个逻辑是这样的，如果执行了 `init_complex_filters()` 就不会执行 `init_simple_filtergraph()`，反之亦然。

`init_complex_filters()` 实际上是对 `init_complex_filter()` 函数进行了一层包装，所以直接讲 `init_complex_filter()` 函数的实现了，流程图如下：

![0-2](init_complex_filtergraph\0-2.jpg)

建议读者用 [ubuntu + clion](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html) 调试下面这条命令，配合着阅读本文更容易理解。

```
ffmpeg -i juren-30s.mp4 -i logo.jpg -filter_complex "[1:v]scale=176:144[logo];[0:v][logo]overlay=x=0:y=0" output.mp4 -y
```

`juren-30s.mp4` 可点击 [此链接](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/encode/juren-30s.mp4) 下载，`logo.jpg` 可点击 此链接 下载。

![0-3](init_complex_filtergraph\0-3.png)

上图中，`avfilter_graph_parse2()` 会解析滤镜字符串 `[1:v]scale=176:144...`，生成对应的数据结构，也可以说是开放了 输入输出的 `AVFilter` 给外部进行连接，因为后面可能还会插入 `trim` 时长裁剪滤镜，等等。命令行用了 `-t 10` 就会插入裁剪滤镜。

输入输出的 `AVFilter` 就是 `AVFilterInOut *inputs, *outputs`，不过在 `init_complex_filtergraph()` 里面还没开始连接 `inputs` 跟 `outputs`，这里只是提取了里面的 `name` 字段的值 `[1:v]` 来确认输入流。

真正连接 `inputs` 跟 `outputs` 是在 `configure_filtergraph()` 函数里，这个后面会提到。

`init_complex_filtergraph()` 函数的重点是调用了 `init_input_filter()` 来绑定 `InputFilter` 跟 `InputStream`，下面我们来学习一下它是如何绑定的。

![0-5](init_complex_filtergraph\0-5.png)

`init_input_filter()` 第一步，就是根据 `in->name` 来确认应该由哪个输入流来输入数据给这个 `InputFilter`。

在此刻，`in->name` 等于 `1:v`，这个 1 是 `input_files[]` 数组的索引，代表第二个输入文件。后面的 `v` 代表这个输入文件里面的视频流，如果文件有多个视频流，只匹配第一个视频流，其他不管。

找到输入流 `ist` 之后，就开始设置相关的属性，如下：

```
ist->discard         = 0;
ist->decoding_needed |= DECODING_FOR_FILTER;
ist->st->discard = AVDISCARD_NONE;
```

这里比较重点，在 `open_input_file()` 打开输入文件的时候，所有的输入流的 `discard` 都是 1，都是丢弃的状态，只有在被需要的时候，输入流才会启用。同理，所有输入流默认是不需要解码的，只有 `decoding_needed` 被设置成非零才会进行解码操作。

接下来就会申请 `InputFilter` 的内存，然后绑定 `InputFilter` 跟 `InputStream`，这两个数据结构是双向绑定的，代码如下：

![0-6](init_complex_filtergraph\0-6.png)

`InputFilter` 只会关联一个 `InputStream`，代表这个滤镜的数据是由那个输入流提供的。而 `InputStream` 会关联多个 `InputFilter`，代表这个输入流的数据需要发送给哪些滤镜。

提醒：上图中把 `InputFilter` 里面的 `frame_queue` 初始化为 8 个 `AVFrame` 大小，这是缓存队列。在未调 `avfilter_graph_config()` 函数正式打开滤镜容器之前，发送给滤镜的数据都会先缓存到 `frame_queue` 队列里面，等到打开了滤镜容器后，就需要把他们全部送给滤镜容器。

因为需要所有的输入流都解码出数据，才会正式打开滤镜容器，在有多个输入流的时候，有些输入流的数据是快一点解码出来的，解码出来就要先缓存在 `frame_queue` 里面。

`InputFilter` 跟 `InputStream` 绑定后的数据结构如下：

![0-7](init_complex_filtergraph\0-7.jpg)

到这里，`init_input_filter()` 函数就讲解完毕了，做个小总结， `init_input_filter()` 的职责就是绑定 `InputFilter` 跟 `InputStream`。

---

现在接着看回去 `init_complex_filtergraph()` 函数后面的代码，如下：

 ![0-8](init_complex_filtergraph\0-8.png)



上图中，注释是初始化了 `OutputFilter` 的数据，不过需要注意是他在循环条件里面没有执行 `cur = cur->next`，而是在循环内部执行的 `cur = cur->next`，应该是有需要的。

不过在 `init_complex_filtergraph()` 函数里面没有绑定 `OutputFilter` 与 `OutputStream`，这两个东西的绑定是在 `open_output_file()` 里面的 `init_output_filter()` 函数进行的。

可以看一下上面的流程图， `open_output_file()` 就在 `init_complex_filtergraph()` 函数后面被调用。

---

`init_complex_filtergraph()` 函数还有一个重点，就是他会释放掉 `graph`，如下：

```
/* this graph is only used for determining the kinds of inputs
 * and outputs we have, and is discarded on exit from this function */
graph = avfilter_graph_alloc();
ret = avfilter_graph_parse2(graph, fg->graph_desc, &inputs, &outputs);

...省略代码...

fail:
    avfilter_inout_free(&inputs);
    avfilter_graph_free(&graph);
```

他的注释写的很清楚，这个 `graph` 只是临时用一下而已，用确定输入输出。后面在 `configure_filtergraph()` 函数里会再调 `avfilter_graph_parse2()` 生成一个新的 `graph` 来完成滤镜的功能。

至此，`init_complex_filtergraph()` 函数的源码分析完毕。

---

`init_complex_filtergraph()` 以及  `open_output_file()` 函数执行完毕之后，整个数据结构的关系如下：

![0-3](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph/0-3.jpg)

提醒：虽然`init_complex_filtergraph()` 里面申请了 `InputFilter` 跟 `OutputFilter` 的内存，但是里面的滤镜上下文（AVFilterContext）还是 NULL，还没开始创建的。

`AVFilterContext` 的创建是在 `configure_filtergraph()` 函数里的，推荐阅读《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》

---

补充：在 `transcode_init()` 函数里会打印哪些输入流 输入到 哪些滤镜里，如下：

![0-9](init_complex_filtergraph\0-9.png)

