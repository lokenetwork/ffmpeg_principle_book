# decode_video解码视频帧—ffmpeg.c源码分析

<div id="meta-description---">decode_video() 不仅仅会对传递进来的 AVPacket 进行解码，如果解码出来数据，就会调 send_frame_to_filters() 发送给滤镜进行处理。</div>

`decode_video()` 是解码视频帧的函数，只在 `process_input_packet()` 里面被调用，用 **Call Hierarchy** 查看，如下：

![1-1](decode_video\1-1.png)

`decode_video()` 函数的定义如下：

```
static int decode_video(InputStream *ist, AVPacket *pkt, int *got_output, int64_t *duration_pts, int eof,
                        int *decode_failed)
```

参数解析如下：

1，`InputStream *ist`，代表解码的是哪个输入流的 `AVPacket`。

2，`AVPacket *pkt`，要进行解码的 `AVPacket`，这个参数实际上有 **三种传参**，如下：

pkt 等于 null，这种情况代表不往解码器发送 `AVPacket`，只是从解码器读 `AVFrame`，这是为了应对发送一个 `AVPacket`，解码器吐出来多个 `AVFrame` 的情况。

pkt 不等于 null，但是它的 size 等于 0，这种情况代表需要 flush 冲刷解码器，把剩余的帧都刷出来，通常只有处理到文件结尾的时候会这么做。

pkt 不等于 null，它的 size 也不等于 0，这是正常的 `AVPacket` 解码，会往解码器发送 `AVPacket`，也会从解码器读 `AVFrame`。

---

3，`int *got_output`，代表解码器是否吐出来了一个 `AVFrame`。

4，`int64_t *duration_pts`，解码器吐出来的 `AVFrame` 的 `duration` 

5，`int eof`，是否解码结束，这个参数不是特别重要，本身就是 pkt 取反计算出来的。

6，`int *decode_failed`，如果是 1，代表解码器内部发生了错误。

---

上面提到 第二个参数 `AVPacket *pkt` 有三种传参。但是 `process_input_packet()` 只有两种传参，如下：

![1-2](decode_video\1-2.png)

这是怎么回事呢？

实际上 `process_input_packet()` 的 NULL 会转成 size = 0 的 `AVPacket` 传给 `decode_video()`，具体怎么做的，再重温有一下  `process_input_packet()`  函数的代码，如下：

![1-3](decode_video\1-3.png)

注意看，`avpkt` 实际上是 `ist->pkt`，只有 `pkt` 参数不是 null 才会进行引用转移。

而且，每次解码完 `avpkt`，就会进行解引用，解引用会释放内存，导致 size 变成 0 ，如下：

![1-4](decode_video\1-4.png)

因此如果 `pkt` 参数是 null，就不会调 `av_packet_ref()` 函数，`avpkt` 的 size 就是 0 ，这样，就传了一个 size 等于 0 的 `AVPacket` 给 `decode_video()` 函数了，如下：

![1-5](decode_video\1-5.png)

---

下面来讲一下 `decode_video()` 函数里面的具体代码。

`decode_video()` 函数一开始创建了 几个局部变量，先介绍一下他们，如下：

```
AVFrame *decoded_frame;
int i, ret = 0, err = 0;
int64_t best_effort_timestamp;
int64_t dts = AV_NOPTS_VALUE;
```

1，`AVFrame *decoded_frame`，指向 `ist->decoded_frame`，用来存储解码出来的 `AVFrame`。

2，`int64_t best_effort_timestamp`，这个变量就是打印一下日志用的。

3，`int64_t dts`，这个变量，代码的作者也不确定它是否有用，如下：

![1-6](decode_video\1-6.png)

他取了 `ist->dts` 来 赋值给 `pkt->dts`，为什么这么做呢？可能是原来的 `pkt->dts` 在一些特殊场景下不准确，但是注释写着可能不需要这么做。

`ist->dts` 的计算方式推荐阅读《[process_input_packet解码封装](https://ffmpeg.xianwaizhiyin.net/ffmpeg/process_input_packet.html)》

---

`decode_video()` 函数接下来就是调 `decode()` 函数进行解码，在解码前后，会调 `update_benchmark()` 对解码时间进行统计，如下：

```
update_benchmark(NULL);
ret = decode(ist->dec_ctx, decoded_frame, got_output, pkt);
update_benchmark("decode_video %d.%d", ist->file_index, ist->st->index);
if (ret < 0)
	*decode_failed = 1;
```

decode() 是一个公共的解码函数，音频，视频解码都是用的它，代码如下：

```
// This does not quite work like avcodec_decode_audio4/avcodec_decode_video2.
// There is the following difference: if you got a frame, you must call
// it again with pkt=NULL. pkt==NULL is treated differently from pkt->size==0
// (pkt==NULL means get more output, pkt->size==0 is a flush/drain packet)
static int decode(AVCodecContext *avctx, AVFrame *frame, int *got_frame, AVPacket *pkt)
{
    int ret;

    *got_frame = 0;

    if (pkt) {
        ret = avcodec_send_packet(avctx, pkt);
        // In particular, we don't expect AVERROR(EAGAIN), because we read all
        // decoded frames with avcodec_receive_frame() until done.
        if (ret < 0 && ret != AVERROR_EOF)
            return ret;
    }

    ret = avcodec_receive_frame(avctx, frame);
    if (ret < 0 && ret != AVERROR(EAGAIN))
        return ret;
    if (ret >= 0)
        *got_frame = 1;

    return 0;
}
```

从上面的代码可以看出来，如果 `pkt` 是 null，就不会调 `avcodec_send_packet()` ，而如果 pkt 的 size 等于 0 ，却回会调 `avcodec_send_packet()` 

---

接下去的代码就没有太多重点了，就是用 `check_decode_result()` 统计一下解码错误率，如果错误率高于 2/3 ，`ffmpeg.exe`转换器就会提示 Conversion failed!，返回值是 69

然后就是统计当前一共解码出来多少帧数据了。

```
ist->frames_decoded++;
```

---

最后调 `send_frame_to_filters()` 把 `AVFrame` 发送给滤镜进行处理，推荐阅读《[send_frame_to_filter滤镜处理](https://ffmpeg.xianwaizhiyin.net/ffmpeg/send_frame_to_filter.html)》

`decode_video()` 函数里面有一段硬件解码的逻辑，如下，推荐阅读《FFmpeg硬件编解码原理》

```
if (ist->hwaccel_retrieve_data && decoded_frame->format == ist->hwaccel_pix_fmt) {
    err = ist->hwaccel_retrieve_data(ist->dec_ctx, decoded_frame);
    if (err < 0)
  		goto fail;
}
ist->hwaccel_retrieved_pix_fmt = decoded_frame->format;
```
