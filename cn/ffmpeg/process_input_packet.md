# process_input_packet解码封装—ffmpeg.c源码分析

<div id="meta-description---">process_input_packet() 是对解码函数 decode_video() ，decode_audio() 的封装。主要功能是 解码第二个参数 const AVPacket *pkt，如果有解码数据出来，就发送给输入流关联的所有入口滤镜。</div>

`process_input_packet()` 的主要功能是 解码第二个参数 `const AVPacket *pkt`，如果有解码数据出来，就发送给输入流关联的**所有入口滤镜**。

函数的定义如下：

```
/* pkt = NULL means EOF (needed to flush decoder buffers) */
static int process_input_packet(InputStream *ist, const AVPacket *pkt, int no_eof)
```

参数解析如下：

1，`InputStream *ist`，要处理的这个 `AVPacket` 是属于哪个输入流的。

2，`const AVPacket *pkt`，要送给解码器的 `AVPacket` 。如果为 NULL，就会 flush 解码器，把解码器的数据都刷出来。

3，`int no_eof`，`no_eof` 是用来控制要不要往输入流绑定的入口滤镜发送 `eof`。当 `pkt` 为 NULL 的时候这个字段才会生效。

当 `pkt` 为 NULL ， `no_eof` 为 1 的时候，`process_input_packet()` 内部会 flush 解码器，把剩余的 `AVFrame` 都刷出来，**但是不会对入口滤镜进行 `eof` 操作**，因为文件会循环，还需要复用之前的滤镜容器。如下：

![1-1](process_input_packet\1-1.png)

![1-2](process_input_packet\1-2.png)

 `no_eof` 设置为 1 只有在文件循环的时候才用到。 

---

`process_input_packet()` 函数有以下重点。

**第一个重点**，`ist->pts`，`ist->dts`，`ist->next_pts`，`ist->next_dts`，这 4 个字段的计算。

`ist->pts` 代表当前流解码到什么时刻了，它不是直接取最后一个的 `AVPacket` 的 `pts`，而是取的上一个 `AVPacket` 的 `pts` 加上 这个 `AVPacket` 的 `duration`。如下：

![1-3](process_input_packet\1-3.png)

`ist->pts` 一开始是赋值为 0 ，然后在 `while (ist->decoding_needed) {...}` 循环里面，`ist->pts` 被赋值为 `ist->next_pts`，如下：

![1-4](process_input_packet\1-4.png)

而 `next_pts` 就是用上一个 `AVPacket` 的 `pts` 加上 `duration` 计算出来的，如下：

![1-5](process_input_packet\1-5.png)

---

`ist->dts` 代表当前流解码的 `dts` 时刻了，可以理解为当前解码的 `dts`，这个字段有两种计算方式。

**1，**`ist->dts` 未进入解码循环之前，是直接取的 `AVPacket` 的 `dts` ，如下：

![1-5-2](process_input_packet\1-5-2.png)

**2，**如果解码器有数据出来，`ist->dts` 就会用上一个 `AVPacket` 的 `dts` 加上 `duration` 计算出来的，如下：

`ist->dts` 的第一次赋值有点复杂，是用 帧率 + B 帧数量计算出来的，如下：

![1-6](process_input_packet\1-6.png)

然后在 `while (ist->decoding_needed) {...}` 循环里面，`ist->dts` 被赋值为 `ist->next_dts`，如下：

![1-4](process_input_packet\1-4.png)

而 `next_dts` 就是用上一个 `AVPacket` 的 `dts` 加上 `duration` 计算出来的，如下：

![1-7](process_input_packet\1-7.png)

`ist->next_pts` 与 `ist->next_dts` ，自然就是下一个的时间。

具体这 4 个字段的计算方式特别复杂，但是好像只有打印日志的时候用了一下。

---

**process_input_packet() 函数的第二个重点，就是 `while (ist->decoding_needed){...}` 解码循环。**

由于往解码器发送一个 `AVPacket`，解码器可能会吐出来多个 `AVFrame`，所以需要一个循环来接受所有的 `AVFrame`。

while 循环里面有 4 个局部变量，如下：

```
while (ist->decoding_needed) {
    int64_t duration_dts = 0;
    int64_t duration_pts = 0;
    int got_output = 0;
    int decode_failed = 0;
    ...省略代码...
}
```

**1，**`int64_t duration_dts`，主要给视频流使用的，默认就是 `pkt->duration` ，如果 `pkt->duration` 是 空，就用帧率来预估，如下：

![1-8](process_input_packet\1-8.png)

**2，**`int64_t duration_pts`，虽然是 `decode_video()` 函数修改的这个字段，但是实际上也是 `pkt->duration`，只是换了一下单位。

**3，**`int got_output`，代表是否从 解码器 读取到 `AVFrame`，也是 `decode_video()` 函数修改的这个字段，如果  `decode_video()` 能从解码器读到 `AVFrame` ，就会把这个 `got_output` 设置为 1。

**4，**`int decode_failed`，解码器是否返回失败。

实际上，`while` 循环里面的解码过程都是调子函数 `decode_audio()`，`decode_video()` 来实现的，推荐阅读《[decode_audio解码音频帧](https://ffmpeg.xianwaizhiyin.net/ffmpeg/decode_audio.html)》《[decode_video解码视频帧](https://ffmpeg.xianwaizhiyin.net/ffmpeg/decode_video.html)》

---

 `while (ist->decoding_needed){...}` 解码循环里面还有一个重点，就是 repeating 变量，这个变量代表在循环里面是不是第二次读取解码器。

第二次读取解码器，是不需要发送 `AVPacket` 的，可以直接传 NULL。

![1-9](process_input_packet\1-9.png)

这是解码函数 `decode()` 的封装规则。

![2-1](process_input_packet\2-1.png)

我们在使用 `avcode_send_packet()` 函数的时候，如果想 flush 解码器，你可以传 `pkt` 等于 `null`，或者 `pkt->size` 等于 0 都可以的，都可以 flush 解码器。

但是对于 `decode()` 函数，他做了一下区别，必须  `pkt->size` 等于 0 才是 flush 解码器， `pkt` 等于 `null` 就不会调 `avcodec_send_packet()` ，所以就不会 flush 解码器，只会不断从解码器读数据，直到没数据可读。

---

**process_input_packet() 函数的第三个重点是它的解码结束逻辑**，如下：

```
/* after flushing, send an EOF on all the filter inputs attached to the stream */
/* except when looping we need to flush but not to send an EOF */
if (!pkt && ist->decoding_needed && eof_reached && !no_eof) {
    int ret = send_filter_eof(ist);
    if (ret < 0) {
        av_log(NULL, AV_LOG_FATAL, "Error marking filters as finished\n");
        exit_program(1);
    }
}
```

如果解码器已经解码结束了，`decode_audio()` 或者 `decode_video()` 函数返回 AVERROR_EOF 就代表解码结束，要想解码结束，必须发一个  `pkt->size` 等于 0 进去。

解码结束就会调 `send_filter_eof(ist)` 来关闭入口滤镜，因为一个输入流可能关联多个入口滤镜，所以每个入口滤镜都会被关闭。

不过如果有文件循环，就不需要关闭入口滤镜。文件循环下 `no_eof` 会是 1，

关于结束处理的整体逻辑，推荐阅读《[FFmpeg转换器转码结束分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/transcode_done.html)》

---

还剩一些 stream_copy 的代码，相对较简单，不进行讲解。

![2-2](process_input_packet\2-2.png)

---

最后，`process_input_packet()` 的返回值是 `!eof_reached`，代表是否解码结束。

```
return !eof_reached
```





