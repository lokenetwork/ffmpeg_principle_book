# FFmpeg转换器转码结束分析—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg转换器转码结束分析,下面的 API 函数都会用一个 AVERROR_EOF 的**错误码**来代表操作结束了。</div>

`ffmpeg.exe` 转换器，在以下 3 种情况下会结束转码，或者结束转封装。

**1，读取到了输入文件的结尾。**

**2，**在命令行对输入文件使用了 `-t 60` 选项，限制只读取 60 秒的输入文件来处理，处理完 60秒的数据就会退出。

**3，**在命令行对输出文件使用了 `-t 60` 选项，限制只输出 60 秒的数据到文件保存，一旦存储够了 60 秒，就会自动退出。

第一种情况是最常见的，第二，第三种情况是通过 `trim` 滤镜 实现的。

本文主要分析**第一种情况**的逻辑，第二第三种请读者自行探索。

---

下面的 API 函数都会用一个 AVERROR_EOF 的**错误码**来代表操作结束了。

**1，**`av_read_frame()`，AVERROR_EOF 代表已经读取到输入文件的结尾。

**2，**`avcodec_send_packet()`，向解码器发送 `AVPacket` 的时候也有可能返回 AVERROR_EOF，通常是因为你之前已经给这个解码器发送过一次 `pkt` == `null` 或者 `pkt.size` == 0 。你第二次再往解码器发送一个 null 的 `AVPacket` 或者 size 等于 0 的 `AVPacket`，`avcodec_send_packet()` 函数就会返回 AVERROR_EOF 

**3，**`avcodec_receive_frame()`，从解码器读取 `AVFrame` 也有可能会返回 AVERROR_EOF，不过前提是，你之前已经给这个解码器发送过一次 `pkt` == `null` 或者 `pkt.size` == 0。发送完这个 null 的 `AVPacket` 之后，一直读解码器，读到没有数据了，就会返回 AVERROR_EOF。

`avcodec_receive_frame()` 函数其实有两个错误码，一个是 `ERROR(EAGAIN)`，一个是 `AVERROR_EOF`，

`ERROR(EAGAIN)`，代表解码器暂时没有数据可读，你要输入更多的 `AVPacket`。

`AVERROR_EOF`，代表解码器解码结束了，没有更多的 `AVPacket` 来输入了。

**4，**`av_buffersink_get_frame_flags()`，从 `buffersink` 出口滤镜读取 `AVFrame` 也有可能会返回 `AVERROR_EOF`，不过前提是已经用 `av_buffersrc_close()` 关闭了所有的 **入口滤镜**。

**5，**`avfilter_graph_request_oldest()`，这个函数是返回 滤镜容器里面还有多少 `AVFrame` 还未读出来，也可能会返回 `AVERROR_EOF`，前提是已经用 `av_buffersrc_close()` 关闭了所有的 **入口滤镜**，或者使用了 trim 滤镜，到时间点就会自动返回 `AVERROR_EOF`

**6，**`avcodec_receive_packet()`，从编码器读取 `AVPacket` 也有可能会返回 AVERROR_EOF，前提是，你之前已经给这个编码器发送过一次 null 的 `AVFrame`。

---

下面就来看一下 `ffmpeg.exe` 转换器是怎么利用上面这些 API 函数的 AVERROR_EOF 返回值的。

首先是 `av_read_frame()` 函数返回 AVERROR_EOF ，会导致哪些分支逻辑被执行？如下：

![1-1](transcode_done\1-1.png)

`av_read_frame()` 函数是封装在 `get_input_packet()` 函数里面的。

可能看到，读取到文件结尾之后，会导致 `process_input_packet()` 函数传一个 `NULL` 的 `AVPacket` 进去。

注意：是该输入文件的**所有输入流**，都会传一个 `NULL` 的 `AVPacket` 进去。

在《[decode_video解码视频帧](https://ffmpeg.xianwaizhiyin.net/ffmpeg/decode_video.html)》里面提过，**`process_input_packet()` 的 NULL 会转成 size = 0 的 `AVPacket` 传给 `decode_video()`**

`size = 0` 的 `AVPacket` 会导致开始冲刷解码器，如下：

![1-2](transcode_done\1-2.png)

上面的 `avcodec_send_packet()` 的返回值也要注意一下，虽然他也会返回 `AVERROR_EOF`，但是在 `ffmpeg.c` 里面是不处理这个状态的，如下：

```
//如果是 AVERROR_EOF 直接跳过。
if (ret < 0 && ret != AVERROR_EOF)
	return ret;
```

---

当 `decode()` 函数返回 `AVERROR_EOF` 之后，会再导致哪些行为呢？如下：

![1-3](transcode_done\1-3.png)

由于 `decode()` 是公共函数，被 `decode_video()` 跟 `decode_audio()` 包起来了，因此会导致 `decode_video()` 跟 `decode_audio()` 的返回值是 `AVERROR_EOF`

---

如果 `decode_video()` 函数返回 `AVERROR_EOF` ，又会导致哪些行为呢？如下：

![1-4](transcode_done\1-4.png)

可以看到会导致 `eof_reached` 变成 1，顺便 break 跳出 `while (ist->decoding_needed){...}` 解码循环。

接着会导致调用 `send_filter_eof()` 函数 关闭该流绑定的所有入口滤镜，如下：

![1-4-2](transcode_done\1-4-2.png)

接着会导致 `process_input_packet()` 返回 0 ，如下：

![1-5](transcode_done\1-5.png)

但是这个 `process_input_packet()` 的返回值，我个人觉得根本没用到，好几个地方即使返回 1，也会转成 0 。

---

虽然 `process_input_packet()` 的返回值没用，但是当它传递 NULL，执行完之后，解码器的所有 `AVFrame` 都已经吐出来了，发送给滤镜进行处理。

并且会把 `ifile->eof_reached` 设置为 1，如下：

![1-6](transcode_done\1-6.png)

因此，**整个解码器的结束流程是**，`av_read_frame()` 返回 EOF，导致解码器吐出来所有的 `AVFrame`，把所有的 `AVFrame` 发生给滤镜之后，再关闭入口滤镜，再把 `ifile->eof_reached` 被设置为 1，代表这个输入文件已经处理完成。

整个流程如下：

![1-7](transcode_done\1-7.jpg)

上图还有一个重点，就是当 `ifile->eof_reached` 被设置为 1 的时候，也会立即返回 `AVERROR(EAGAIN)`。

当 `process_input()` 函数返回  `AVERROR(EAGAIN)`，这样就会导致 `transcode_step()` 不执行 `reap_filters()` ，直接返回 0。如下：

![1-8](transcode_done\1-8.png)

---

当输入流已经处理完成，解码器已经吐出来所有 `AVFrame`，`ifile->eof_reached` 被设置为 1，并且入口滤镜已经用 `send_filter_eof()` 关闭了。

一旦关闭入口滤镜，`av_buffersink_get_frame_flags()` 或者 `avfilter_graph_request_oldest()` 就有机会返回 `AVERROR_EOF`。

先来看一下 `av_buffersink_get_frame_flags()` 函数返回 AVERROR_EOF 的行为，如下：

`av_buffersink_get_frame_flags()` 是封装在 **`reap_filters()`** 里面的。

![1-9](transcode_done\1-9.png)

可以看到  `av_buffersink_get_frame_flags()` 函数返回 AVERROR_EOF 会导致 `do_video_out()` 传递 NULL，但是实际调试后发现，这个 NULL 会导致冲刷编码器，没有导致 传递 NULL `AVPacket` 给编码器

传递 NULL `AVPacket` 刷新编码器，会在 `flush_encoders()` 函数进行。

因此  `av_buffersink_get_frame_flags()` 函数返回 AVERROR_EOF 的行为**我们可以不用关注**。

补充：`do_video_out(of, ost, NULL);` 不是用来刷编码器的，**应该是用来刷帧率变化剩下的帧的**，，如下：

```
//next_picture 等于 NULL
if (!next_picture) {
    //end, flushing
    nb0_frames = nb_frames = mid_pred(ost->last_nb0_frames[0],
                                        ost->last_nb0_frames[1],
                                        ost->last_nb0_frames[2]);
}
```

---

`avfilter_graph_request_oldest()` 函数返回 `AVERROR_EOF` 的情况却是要特别注意。

因为之前关闭了入口滤镜，所以当容器没有数据可读的时候， `avfilter_graph_request_oldest()` 就会返回 `AVERROR_EOF`，如下：

![2-1](transcode_done\2-1.png)

`avfilter_graph_request_oldest()` 返回 `AVERROR_EOF`，会导致两个行为。

**1，**`reap_filter(1)`，参数是 1，所以会刷完 **帧率变化剩下的帧**，但是不会冲刷编码器。

![2-2](transcode_done\2-2.png)

**2，**`close_output_stream(graph->outputs[i]->ost)`，设置 输出流的 `finished` 状态。

```
ost->finished |= ENCODER_FINISHED;
```

**这个  `finished` 状态 特别重要。**

在之前的流程图中，可以看到，`ffmpeg.exe` 会一直循环执行 `transcode_step()` ，如下：

![1-1](https://ffmpeg.xianwaizhiyin.net/ffmpeg/transcode_step/1-1.jpg)

`ffmpeg.exe` 转换器什么情况会跳出上面这个 `while` 循环呢？

答：是通过输出流的 `finished` 状态判断的，如下：

![2-3](transcode_done\2-3.png)

---

到这里，ffmpeg.exe 的结束处理逻辑已经走了 80%，已经跳出来了  `while (!received_sigterm)`  循环。

但是，此时编码器还是没有进行冲刷的，在 `do_video_out(of, ost, NULL);` 的时候并没有对编码器进行冲刷。

**编码器的冲刷是在跳出 `while (!received_sigterm)`  循环之后操作的**，如下：

![2-4](transcode_done\2-4.png)

`flush_encoder()` 函数不太复杂，所以不讲解了。

在最后，还是进行一个 `av_write_trailer()` 操作，写入文件的尾部信息，如下：

![2-5](transcode_done\2-5.png)

还有一些释放各种资源的操作就不讲解了。

---

`ffmpeg.exe` 转换器转码结束分析完毕。
