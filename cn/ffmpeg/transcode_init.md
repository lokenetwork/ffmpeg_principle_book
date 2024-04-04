# transcode_init转码前的初始化—ffmpeg.c源码分析

<div id="meta-description---">transcode_init() 函数是 ffmpeg.exe 转换编码格式，或者转换封装格式之前的操作，主要是一些初始化的操作。</div>

`transcode_init()` 函数是 `ffmpeg.exe` 转换编码格式，或者转换封装格式之前的操作，主要是一些初始化的操作。

`transcode_init()` 在函数调用中的位置如下：

![1-1](transcode_init\1-1.jpg)

`transcode_init()` 里面 全都是 `for` 循环，一共有 8 个 `for` 循环，下面就一个一个来讲解。

**第一个 for 循环：赋值 source_index**，如下：

![1-2](transcode_init\1-2.png)

这段逻辑其实不是特别重要，在**简单滤镜**场景下不会跑进去，因为如果是简单滤镜，`source_index` 会赋值为大于等于 0 。

上图有两个 if 判断。

1，`source_index` 必须等于 -1，只有复杂滤镜才会设置为 -1。

2，`fg->nb_inputs` 必须等于 1，只有一个输入流。

因此我个人猜测，这段代码，是处理复杂滤镜场景里面的，一个输入流，拆出来多个输出流的场景，具体命令我也没找到，后面补充。

---

**第二个 for 循环：模拟帧率**，如下：

推荐阅读《FFmpeg录像模拟直播分析》

```
/* init framerate emulation */
for (i = 0; i < nb_input_files; i++) {
    InputFile *ifile = input_files[i];
    if (ifile->rate_emu)
        for (j = 0; j < ifile->nb_streams; j++)
        	input_streams[j + ifile->ist_index]->start = av_gettime_relative();
}
```

---

**第三个 for 循环：用 init_input_stream 初始化输入流**，这是本文最重要的一个 for 循环逻辑，因为它在简单场景下会跑进去。

`init_input_stream()` 函数的逻辑比较简单，主要是设置 `get_format` 回调函数，`get_buffer2` 回调函数，如下：

```
ist->dec_ctx->opaque                = ist;
ist->dec_ctx->get_format            = get_format;
ist->dec_ctx->get_buffer2           = get_buffer;
```

上面的**`opaque`** 是传递给回调函数的参数。这两个回调函数的作用如下：

**1，**`get_format`，这是一个回调函数，在 `avcodec_open2()` 打开的解码器的时候会调用 `get_format()`，根据 `get_format` 的返回值决定解码器输出哪种 像素格式，**对于软解码来说，你通常不需要设置这个回调函数**，解码器会选择质量最好的像素格式进行输出。

解码器支持的像素格式数组的第一个就是质量最好的，解码器里面的 `pix_fmts` 数组是按照质量优劣排列的。

这个回调函数在 `ffmpeg.c` 里面的实现是 `get_format()` 函数，主要是应对硬件解码场景的，推荐阅读《FFmpeg硬件编解码原理》。

**2，**`get_buffer2`，这个回调函数在  `ffmpeg.c` 里面的实现 也是为了应对硬件解码场景的，软解码不需要设置这个回调。

`get_buffer2` 回调在 `ffmpeg.c` 里面的实现是 `get_buffer()` 函数，如下：

```
static int get_buffer(AVCodecContext *s, AVFrame *frame, int flags)
{
    InputStream *ist = s->opaque;

    if (ist->hwaccel_get_buffer && frame->format == ist->hwaccel_pix_fmt)
        return ist->hwaccel_get_buffer(s, frame, flags);

    return avcodec_default_get_buffer2(s, frame, flags);
}
```

你不设置这个 `get_buffer2` ，它解码器默认也是调的 `avcodec_default_get_buffer2` 来获取数据。

---

**`init_input_stream()` 函数最后会打开解码器，如下：**

```
if ((ret = avcodec_open2(ist->dec_ctx, codec, &ist->decoder_opts)) < 0) {
    ...省略代码...
}
```

`init_input_stream()` 函数的流程如下：

![1-1-2](transcode_init\1-1-2.jpg)

这里必须介绍一下 clion 的 **Call Hierarchy** 功能，它可以看到 函数的调用层级，如下：

![1-1-3](transcode_init\1-1-3.png)

---

**第四个 for 循环：初始化 stream copy 场景的输出流**，代码如下：

第四个 for 循环其实也不是特别重要，因为他只有在 stream copy 的场景会跑进去，如果命令行没使用 copy 选项，就会进行编解码，就不会跑进去这里。

注意那个条件 `output_streams[i]->stream_copy`，就是判断是不是 stream copy 的场景。

相关的介绍推荐阅读《init_output_stream_wrapper初始化输出流》

![1-3](transcode_init\1-3.png)

---

![1-4](transcode_init\1-4.png)

后面的 4 个 for 循环都不是特别重要，那个 programs 一般是 TS 流会有，no streams 的场景比较罕见，所以可以先不管这 4 个 for 循环。

----

不过 `transcode_init()` 函数最后还有一个重点，就是原子类型的操作，如下：

```
atomic_store(&transcode_init_done, 1);
```

`transcode_init_done` 变量是一个原子类型。

```
static atomic_int transcode_init_done = ATOMIC_VAR_INIT(0);
```

这是 C11 标准的原子类型，是线程安全的。

