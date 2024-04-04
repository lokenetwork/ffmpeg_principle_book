# init_output_stream初始化输出流—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

`init_output_stream()` 是一个公共的函数，无论是音频，还是视频的输出流的初始化，都是通过它来完成的。

`init_output_stream()` 上面还会套一个 `wrapper` ，主要是做一些简单的封装，例如已经初始化了，就直接返回，代码如下：

![1-1](init_output_stream\1-1.png)

---

音频 `OutputStream` 的初始化主要有两个地方。

**1，**如果是 stream copy，不进行编解码，就是在 `transcode_init()` 里面如下：

![1-2](init_output_stream\1-2.png)

从上图可以看到，如果不进行编解码，音频，视频的输出流，都是在 `transcode_init()` 里面初始化的。

**2，**滤镜模式，进行了编解码，就是在 `reap_filter()` 里 初始化音频的输出流的，如下：

![1-3](init_output_stream\1-3.png)

音频的输出流可以在未从滤镜读取到 `AVFrame` 的时候就开始初始化，而视频的输出流的初始化，需要从滤镜拿到 `AVFrame` 之后才能初始化，视频的初始化是在 封装在`do_video_out()` 函数里面的。

至于为什么音频输出流需要提前初始化，可以看一下他的注释，我没细看。

---

`init_output_stream()` 函数其实没有太多的重点，主要就是设置编码器参数，然后打开编码器，再设置一下 `OutputStream` 的一些字段，就初始化完成了。

不过 `OutputStream` 有一个字段特别重要，就是 `initialized` ，如下：

![1-4](init_output_stream\1-4.png)

这个 `initialized` 特别重要，只有输出文件里面的所有输出流，包括音频跟视频流，他们的 initialized 都是 1，才能调 `avformat_write_header()` 函数写入头部信息，

因为 `init_output_stream()` 会对 `AVStream` 设置一些信息，例如编码信息等等。

必须设置完这些信息，`initialized` 全部都是 1，才能调 `avformat_write_header()`。

因此，如果音频输出流没初始化完成，那视频流的 `AVPacket` 就不能写入文件，必须先写到队列缓存下来，如下：

![1-5](init_output_stream\1-5.png)

---

`init_output_stream()` 函数的整体流程图如下：

![1-6](init_output_stream\1-6.jpg)

