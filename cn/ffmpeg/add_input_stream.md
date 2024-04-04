# add_input_stream添加输入流—ffmpeg.c源码分析

<div id="meta-description---">add_input_stream() 函数是一个添加输入流的函数，它会把文件里面的输入流全部添加进去 input_streams 数组。而 input_streams 数组是一个全局变量，包含了所有输入文件里面的所有输入流。</div>

`add_input_stream()` 函数是一个添加输入流的函数，它会把文件里面的输入流全部添加进去 `input_streams` 数组。而 `input_streams` 数组是一个全局变量，包含了所有输入文件里面的所有输入流。

```
nputStream **input_streams = NULL;
int        nb_input_streams = 0;
```

你在二次开发 `ffmpeg.exe` 的时候，可以用 `input_streams` 全局变量来获取到所有的**输入流**。

在学习 `add_input_stream()` 函数之前，需要先了解一个新的数据结构 `struct InputStream`，推荐阅读《[InputStream数据结构分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/struct_inputstream.html)》

---

`add_input_stream()` 函数的流程图如下：

![1-1](add_input_stream\1-1.jpg)

`add_input_stream()` 函数的逻辑不算太复杂，只有两三个不太容易看到的地方，本文省略里面的硬件加速的代码，初学者不需要看那段代码，可以后续再学。

----

`add_input_stream()` 函数的重点代码如下：

![1-2](add_input_stream\1-2.png)

上图中的 `skip_frame` 的逻辑不用管，如果命令行没使用 `-discard` 选项，这个逻辑不会用到。

`ffmpeg.c` 里面太多场景的逻辑了，如果你在看源码的时候，各个场景都考虑，会容易迷失在纷繁复杂的分支里面，所以建议直接用下面这一句最简单的命令来调试，弄懂这一个简单的场景先。

```
ffmpeg -i juren.mp4 juren.flv
```

接下来的重点就是流属性 `discard` 设置为丢弃，因为即便 mp4 文件里面有多个视频流，默认只会选取最好的一个流作为输入。

然后把 `OptionsContext` 的一些字段，赋值到 `InputStream` 里面。

---

接下来就是确定解码器，如下：

还要把 `OptionsContext` 里面的解码器参数赋值给 `ist->decoder_opts` 。

```
ist->dec = choose_decoder(o, ic, st);
ist->decoder_opts = filter_codec_opts(o->g->codec_opts, ist->st->codecpar->codec_id, ic, st, ist->dec);

```

然后就是申请解码器实例的内存，还有把容器层的解码参数 par 赋值给解码器实例，如下：

![1-3](add_input_stream\1-3.png)

par 变量是从 `st->codecpar` 里面来的，是容器里面的流的解码器的信息。

注意，在 `add_input_stream()` 里面是没有**打开解码器**的，所以  `ist->decoder_opts`  还没有被使用掉。打开解码器是在 `init_input_stream()` 函数里面做的。

---

接下来就是 一个 switch case 的分支，这是处理不同的数据流的逻辑，设置解码器的帧率之类的，比较简单。

![1-4](add_input_stream\1-4.png)

在最后还会把解码器的参数 复制回去给 容器层，如下：

我也不知道为什么要这么做， `ist->dec_ctx` 的参数大部分都是用 `avcodec_parameters_to_context()` 从容器层复制过来的，现在又复制回去。可能是有些字段被修改了，例如那个帧率之类的。

复制回去之后，好像也没地方需要用到这个 par 的内容，我也不清楚。

```
ret = avcodec_parameters_from_context(par, ist->dec_ctx);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error initializing the decoder context.\n");
    exit_program(1);
}
```

---

总结，`add_input_stream()` 做的事情其实并没有太复杂，主要就两件事情。

1. 申请 新的`InputStream`，把 `OptionsContext` 的一些参数丢给 `InputStream`，主要调 `MATCH_PER_STREAM_OPT()` 来实现。
2. 申请解码器context，做一些参数赋值。



