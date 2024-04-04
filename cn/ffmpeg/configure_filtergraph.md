# configure_filtergraph配置滤镜容器—ffmpeg.c源码分析

<div id="meta-description---">当所有的 InputFilter 都初始化完成，InputFilter 里面的 format 不等于 -1 就是初始化完成，就会调 configure_filtergraph() 函数来配置与打开滤镜容器。configure_filtergraph() 会把所有的 InputFilter 与 OutputFilter 连接在一起。</div>

之前在 `init_simple_filter()` 里面已经对滤镜进行了基础的初始化，但是还有些东西没处理，`avfilter_graph_config()` 在 `init_simple_filter()` 里面也没调用，在《[FFmpeg滤镜API](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/filter_api.html)》一章中，我们知道打开滤镜容器必须调 `avfilter_graph_config()`。

而 `configure_filtergraph()` 就是配置跟打开滤镜容器的函数，这其实是一个通用函数，无论是 简单滤镜，还是 复杂滤镜，都是使用这个 `configure_filtergraph()`  函数来配置滤镜的。

**但是本文主要侧重于简单滤镜场景下的分析。**

---

下面来逐步分析 `configure_filtergraph()`  函数的重点代码。

**第一个重点：graph_desc 变量的值是什么？**

答：在简单场景下，就是命令行没有使用滤镜，那 `graph_desc` 就是 `null` 字符，或者 `anull` 字符，一个是视频空滤镜，一个是音频空滤镜，如下：

![1-1](configure_filtergraph\1-1.png)

如果是 **非简单场景**，例如命令行使用了滤镜，那 `graph_desc` 就是命令行那个滤镜字符串，如下：

```
./ffmpeg -re -i juren.mp4 -re -i juren.mp4 -re -i juren.mp4 -re -i juren.mp4 
-filter_complex 
"nullsrc=size=640x480[base];
[0:v]setpts=PTS-STARTPTS,scale=320x240[upperleft];
[1:v]setpts=PTS-STARTPTS,scale=320x240[upperright];
[2:v]setpts=PTS-STARTPTS,scale=320x240[lowerleft];
[3:v]setpts=PTS-STARTPTS,scale=320x240[lowerright];
[base][upperleft]overlay=shortest=1[tmp1];
[tmp1][upperright]overlay=shortest=1:x=320[tmp2];
[tmp2][lowerleft]overlay=shortest=1:y=240[tmp3];
[tmp3][lowerright]overlay=shortest=1:x=320:y=240"
-c:v libx264 output.mp4
```

![1-2](configure_filtergraph\1-2.png)

---

**第二个重点：设置 scale_sws_opts 自动插入 scale 滤镜**

```
if (strlen(args))
    args[strlen(args)-1] = 0;
fg->graph->scale_sws_opts = av_strdup(args);
```

在简单滤镜场景下，之前在命令行的参数，会这样被使用，

![1-3](configure_filtergraph\1-3.png)

不过最后一个操作，我没看懂，好像是写错了。

![1-4](configure_filtergraph\1-4.png)

上图的这个 args 没有被使用。

---

**第三个重点：解析命令行的滤镜语法，链接输入输出滤镜，以及打开滤镜**

![1-5](configure_filtergraph\1-5.png) 

先来讲一下解析命令行的滤镜语法，如下：

```
./ffmpeg -re -i juren.mp4 -re -i juren.mp4 -re -i juren.mp4 -re -i juren.mp4 
-filter_complex 
"nullsrc=size=640x480[base];
[0:v]setpts=PTS-STARTPTS,scale=320x240[upperleft];
[1:v]setpts=PTS-STARTPTS,scale=320x240[upperright];
[2:v]setpts=PTS-STARTPTS,scale=320x240[lowerleft];
[3:v]setpts=PTS-STARTPTS,scale=320x240[lowerright];
[base][upperleft]overlay=shortest=1[tmp1];
[tmp1][upperright]overlay=shortest=1:x=320[tmp2];
[tmp2][lowerleft]overlay=shortest=1:y=240[tmp3];
[tmp3][lowerright]overlay=shortest=1:x=320:y=240"
-c:v libx264 output.mp4
```

上面的 `[0:v]` 之类的标记，是 `ffmpeg.exe` 转换器自己定义的语法，实际上如果你自己写代码调 `avfilter_graph_parse2()` API函数，你可以用 `[a:vvv]` 来表示第一个文件的视频流，这也是可以的，**这只是一个标示**，这个 `[a:vvv]`  标示会被解析成 `AVFilterInOut` 结构，然后你就可以创建一个 `buffer` 入口滤镜来连接这个 `AVFilterInOut` 。

**重点：对于 `avfilter_graph_parse2()`  API 函数来说，这个 `[0:v]` 标记语法是可以自定义的，定义成怎样的都可以，里面的 0 跟 v 代表什么完全是由你自己的代码决定。**

`avfilter_graph_parse2()`  执行完之后，`[0:v]` ~ `[3:v]` 就会 被解析成 4 个输入的 `AVFilterInOut`，相当于开放了 4 个入口给上层代码来链接。

```
if ((ret = avfilter_graph_parse2(fg->graph, graph_desc, &inputs, &outputs)) < 0)
	goto fail;
```

ffmpeg.exe 转换器链接 4 个入口的上层代码就是 `configure_input_filter()` 函数，如下：

    for (cur = inputs, i = 0; cur; cur = cur->next, i++)
        if ((ret = configure_input_filter(fg, fg->inputs[i], cur)) < 0) {
            avfilter_inout_free(&inputs);
            avfilter_inout_free(&outputs);
            goto fail;
        }

链接输出的代码如下，因为只有一个输出流，所以只有一个输出的 `AVFilterInOut`，下面的循环只会执行一次。

```
    for (cur = outputs, i = 0; cur; cur = cur->next, i++)
        configure_output_filter(fg, fg->outputs[i], cur);
```

`configure_input_filter()` 与 `configure_output_filter()` 函数的内部实现不是特别复杂，主要是创建滤镜，然后用 `avfilter_link()` 把滤镜链接起来，所以不讲解他们内部的代码，滤镜相关的 API 推荐阅读《[FFmpeg滤镜API](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/filter_api.html)》一章。

整个链接过程如下图（只画出了一个输入 AVFilterInOut，简单滤镜的场景 ）：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/ffmpeg_c-10-6.png)

`ffmpeg.exe` 转换器这种使用滤镜的方式特别复杂，如果是单纯用  `avfilter_graph_parse2()`  API 函数，其实有更简洁的用法，是不需要解析 `[0:v]` 标记的，推荐阅读《[FFmpeg的scale滤镜介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/scale.html)》

**做一下小总结，之前在 `init_simple_filter()` 里面只是对滤镜进行了基础的初始化，创建了 输入 `InputFilter`，输出 `OutputFilter` 的内存，但是里面的 `AVFilterContext *filter` 还是 NULL。**

**`configure_filtergraph()` 执行完之后，`InputFilter` 的 `filter` 就指向了有效的内存，不是NULL。而且所有的输入输出滤镜都通过 `avfilter_link()` 链接起来了。**

扩展知识：它里面清空一个字符的语法是把第一个字节置为 0 ，如下：

```
args[0] = 0; 
```

---

**第四个重点：写入 frame_queue队列的数据，冲刷滤镜容器**

![1-6](configure_filtergraph\1-6.png)

在未打开滤镜容器之前，发送给滤镜的数据都会先缓存到 `frame_queue` 队列里面，现在打开了滤镜容器，就需要把他们全部送给滤镜容器。

不过下面的 冲刷滤镜，我没看懂是什么操作，虽然转码过程中，如果 `AVFrame` 中途变化了，会重新配置滤镜，但是应该在重新配置滤镜前就进行冲刷，配置完再冲刷是为了啥？

---

至此，`configure_filtergraph()` 函数的源码分析完毕。

这个函数异常复杂，建议多调试几次源代码，但是这个 `configure_filtergraph()` 函数又封装得特别好，有时候对 `ffmpeg.exe` 二次开发，你甚至可以不去理解`configure_filtergraph()` 的内部实现，只需要它执行完之后，输入流绑定的 `InputFilter` 跟 输出流绑定的 `OutFilter` 都配置完成了。

你只需要把 `AVFrame` 往 `InputFilter`丢，然后从 `OutFilter` 读 `AVFrame` 即可。

