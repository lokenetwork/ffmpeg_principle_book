# transcode_step转码总函数—ffmpeg.c源码分析

<div id="meta-description---">transcode_step() 函数是 ffmpeg.exe 转换编码格式，或者转换封装格式的总函数。transcode_step() 内部每次都会读取一个AVPacket，发送给解码器进行解码，解码器可能会输出 0个 ~ N个 AVFrame，然后把 AVFrame 发送给滤镜容器，如果滤镜容器有数据能出来，就会接着发送给编码器进行编码，如果编码器有 AVPacket 数据可以出来，就把 AVPacket 写入文件保存。</div>

`transcode_step()` 函数是 `ffmpeg.exe` 转换编码格式，或者转换封装格式的**总函数**，它在函数调用中的位置如下：

![1-1](transcode_step\1-1.jpg)

`transcode_step()` 内部每次都会读取一个`AVPacket`，发送给解码器进行解码，解码器可能会输出 0个 ~ N个 `AVFrame`，然后把 `AVFrame` 发送给滤镜容器，如果滤镜容器有数据能出来，就会接着发送给编码器进行编码，如果编码器有 `AVPacket` 数据可以出来，就把 `AVPacket` 写入文件保存。

这就是 `transcode_step()` 总函数的内部逻辑，不过有几点需要注意的。

**1，**往解码器发送一个 `AVPacekt` ，解码器可能没数据输出，这样就会回到 `while` 循环，再次调  `transcode_step()` 读第二个 `AVPacekt` 往解码器丢。

**2，**往滤镜容器发送一个 `AVFrame`，滤镜容器不一定就立马有数据可读，没数据可读，又会回到 while 循环。

**3，**往编码器发送一个 `AVFrame`，编码器不一定就立即输出 `AVPacekt`，所以又会回到 `while` 循环，再次调  `transcode_step()` 解码出第二个 `AVFrame` 往编码器丢。

这个函数名称之所以有 step，step 就是一步的意思，每次只从文件读取一个 `AVPacekt`，只处理一个 `AVPacket`

---

下面一起来学习一下 `transcode_step()` 转码总函数的代码，重点如下：

![1-1-2](transcode_step\1-1-2.png)

`choose_output()` 函数的作用是在所有输出流（OutputStream）中选出最合适的一个。 代码如下：

![1-1-2-1](transcode_step\1-1-2-1.jpg)

可以看到，会优选选择未初始化的 `OutputStream`，如果所有输出流都初始化了，就选择时间最短的那个输出流。最短时间通过 `ost->st->cur_dts` 来判断，例如音频流已经输出了2分钟了，视频流才输出1分钟，`choose_output()` 函数就会返回 视频流。

`ost->initialized` 在出口滤镜输出 `AVFrame` 的时候就会初始化，被赋值为 1，注意不是在解码器输出 `AVFrame` 的时候，而是出口滤镜输出 `AVFrame` 的时候才会初始化输出流，因为这时候才知道具体要编码的像素格式等信息。（不过对于音频输出流，会提前进行初始化）

```
//init_output_stream 函数的代码
ost->initialized = 1;
```

![1-1-2-2](transcode_step\1-1-2-2.png)

`ost->inputs_done` 代表没有找到输出流对应的输入流，通常不会没找到，所以通常一直都是 0

---

第二个重点是三个 `if` 判断，这 3 个 `if` 判断分别是处理不同的场景的，如下：

![1-13](transcode_step\1-1-3.png)

**第一个 `if` 条件，**的代码如下：

```
if (ost->filter && !ost->filter->graph->graph) {
    if (ifilter_has_all_input_formats(ost->filter->graph)) {
        ret = configure_filtergraph(ost->filter->graph);
        if (ret < 0) {
            av_log(NULL, AV_LOG_ERROR, "Error reinitializing filters!\n");
            return ret;
        }
    }
}
```

上面这段代码比较复杂，我也是过了几个月才明白它的场景，

这个 `if` 我个人认为**很有可能是多余的**，我在简单滤镜以及复杂滤镜的场景下都测了，没有跑进去。

上面的条件是 `ost->filter` 不为 NULL，并且还未创建 `graph` 滤镜容器，这两个条件有可能成立。但是这里的 `ifilter_has_all_input_formats()` 不可能同时成立。

补充： `ifilter_has_all_input_formats()` 函数是判断所有的 `InputFilter::format` 不为 `-1` 就成立。

因为 `transcode_step()` 函数后面的代码，只要解码出一个 `AVFrame`，就会调 `ifilter_send_frame()` 函数来把这个 `AVFrame` 发送给 `InputFilter`，而 `ifilter_send_frame()` 会调 `ifilter_parameters_from_frame()` 来把 `InputFilter::format` 设置为非 `-1` 的值。

当所有的 `InputFilter::format` 都设置为非 `-1` 的时候， `ifilter_send_frame()` 里面就会调 `configure_filtergraph()` 来创建 `graph` 滤镜容器了。

如果已经创建了 `graph` 滤镜容器，下次再循环跑进去上面的 `if`，`!ost->filter->graph->graph` 就不等于真了。

所以我观察下来下面上面这 3 个条件不会同时成立，所以我个人判断，上面的这点 `if` 代码可能是多余的，具体我后面在 FFmpeg 社区发个邮件问一下再补充。

---

**第二个 `if` 条件，从滤镜容器选出最合适的输入流**，如下：

```
 if (ost->filter && ost->filter->graph->graph) {
        /*
         * 省略注释
         */
        if (av_buffersink_get_type(ost->filter->filter) == AVMEDIA_TYPE_AUDIO)
            init_output_stream_wrapper(ost, NULL, 1);

        if ((ret = transcode_from_filter(ost->filter->graph, &ist)) < 0)
            return ret;
        if (!ist)
            return 0;
    }
```

上面的代码主要是调用了 `transcode_from_filter()` 函数，确定应该从哪个输入流读取 `AVPacekt`。

具体的算法，是用 `av_buffersrc_get_nb_failed_requests()` 获取每个 `buffer`入口滤镜的失败次数，选出失败次数最多的入口滤镜，最后因为入口滤镜绑定了输入流。所以最合适的输入流也就确定了。

---

**第三个 `if` 条件，当还未打开滤镜容器的时候，选一个从未解码出 `AVFrame` 的输入流来进行处理**，如下：

```
else if (ost->filter) {
    int i;
    for (i = 0; i < ost->filter->graph->nb_inputs; i++) {
        InputFilter *ifilter = ost->filter->graph->inputs[i];
        if (!ifilter->ist->got_output && !input_files[ifilter->ist->file_index]->eof_reached) {
            ist = ifilter->ist;
            break;
        }
    }
    if (!ist) {
    	ost->inputs_done = 1;
    	return 0;
    }
} 
```

上面的 `got_output` 字段，代表这个输入流是否已经解码出来 `AVFrame` 了，解码出来 `AVFrame` 就可以配置对应的入口滤镜的参数。

当所有的**入口滤镜**都配置完成（format 不等于 -1），就可以成功打开滤镜容器，函数的调用流程如下：

![1-1-7](transcode_step\1-1-7.jpg)

关于 `configure_filtergraph` 函数的介绍，推荐阅读《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》。

---

**第四个 `else` 条件**，是最后的保留策略，直接用输出流对应的输入流，如下：

```
else {
    av_assert0(ost->source_index >= 0);
    ist = input_streams[ost->source_index];
}
```

这里的逻辑是只要你命令行使用了 `-c copy` 就会跑进去。因为 `-c copy` 是不进行编解码操作，所以不会使用到滤镜的逻辑，因此 `ost->filter == NULL`

---

做个阶段性的小总结，`transcode_step()` 内部选出输入流的策略只有两个需要关注的地方，就是 第二 跟 第三 个条件。

1，第二个条件是滤镜容器已经打开了，所以从 `transcode_from_filter()` 选出输入流来进行处理。

2，第三个条件是，只要有一个入口滤镜未配置好，也就是 format 等于 -1 的时候，滤镜容器就不能打开。这时候，就选出一个从未解码出 `AVFrame` 的输入流出来，进行处理。

**所以，无论你是用简单滤镜，还是复杂滤镜，`ffmpeg.exe` 它都会先跑到 第三个条件，把所有的输入流都至少解码出一个 `AVFrame`。然后打开滤镜容器，这样才会跑进去第二个条件。**

---

`transcode_step()` 转码总函数的流程图如下：

![1-2](transcode_step\1-2.jpg)

从上图可以看到，选出输入流之后，就会调 `process_input()` 从输入流关联的输入文件里面读取 `AVPacekt`。

`process_input()` 函数的主要逻辑是，读取一个 `AVPacekt`，然后往解码器丢，解码器可能不会出数据，也可能会出来多个 `AVFrame`，无论出来多少个 `AVFrame`，都调 `send_frame_to_filters()` 把它们往入口滤镜发送。

`process_input()`只会往解码器发送一个 `AVPacket`，但是会不断往解码器读 `AVFrame`，直到解码器返回 EAGAIN。

而最后的 `reap_filter()` 函数，就负责从出口滤镜里 读取 `AVFrame`，然后发送给编码器编码，最后保存进去文件。

更详细的介绍，推荐阅读《[process_input处理输入文件](https://ffmpeg.xianwaizhiyin.net/ffmpeg/process_input.html)》《[reap_filter收割滤镜](https://ffmpeg.xianwaizhiyin.net/ffmpeg/reap_filters.html)》
