# FFmpeg读取文件内容AVPacket—FFmpeg API教程

<div id="meta-description---">本文主要介绍 AVPacket 结构体，av_read_frame，av_packet_unref,av_packet_free 函数的使用</div>

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avpacket )，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

------

跟 **读取文件内容** 相关的结构体如下：

1，`AVPacket`，一个管理压缩后的媒体数据的结构，它本身不包含压缩的媒体数据，而是通过 data 指针指向媒体数据。

这里面的媒体数据通常是一帧视频的数据，或者一帧音频的数据。但是也有一些特殊情况，这个 AVPacket 的 data 是空的，只有 side data 的数据。side data 是一些附加信息。

------

跟 **读取文件内容** 相关的函数如下：

1，`av_packet_alloc`，初始化一个 `AVPacket`。

2，`av_read_frame`，从 `AVFormatContext` 容器里面读取一个 `AVPacket`，需要注意，虽然函数名是 `frame`，但是读取的是 `AVPacket`。

3，`av_packet_unref`，减少 `AVPacket` 对 编码数据的引用次数。减到 0 会释放 编码数据的内存。

3，`av_packet_free`，释放 `AVPacket` 自身的内存。里面会调 `av_packet_unref`

------

下面就结合本文代码来理解上面这些函数跟 `AVPacket` 的使用。请看下图：

![avpacket-1-1](avpacket\avpacket-1-1.png)

注意：`av_read_frame` 的返回值有很多种情况。

1，成功读取到一个 `AVPacket` 。

2，读取到文件末尾。

3，网络流有问题，或者本地硬盘坏了。导致 `AVFormatContext` 里面的 `pb->error` 有异常。

我上面的代码没有处理这些情况，我假设他会成功读取到 一个 `AVPacket` 。这些情况 在 `ffplay.c` 里面都被正确处理。请参考  `ffplay.c` 写出更健壮的代码。

------

上面的代码只读了一个 **AVPacket**。

从上图代码可以看出，我用 `av_packet_alloc` 初始化了一个 `AVPacket` ，那是不是用 `av_read_frame` 读一次数据，就要调一次 `av_packet_alloc` 呢？

不是，因为 `AVPacket` 他本身是没有编码数据的，他只是管理编码数据，也就是说 `av_packet_alloc` 的时候，`AVPacket` 里面还没有编码数据的，是后面通过 `av_read_frame`把 `AVPacket` 里面的 `buf` 引用指向了编码数据。

如果你要调多次 `av_read_frame`，只需要先用 `av_packet_unref` 消除 `AVPacket` 里面对之前的编码数据的引用即可。只有最后用不到 `AVPacket` 的时候，才需要调 `av_packet_free` 来释放 `AVPacket` 的内存。

------

上面的代码运行情况如下：

![avpacket-1-2](avpacket\avpacket-1-2.png)

`AVPacket` 里面有一个 `stream_index` 字段，代表这个 `AVPacket`  属于哪个流，那我们怎么知道这是一个音频包，还是视频包？在下面的字段里面：

```
fmt_ctx->streams[0]->codecpar->codec_type
```

我代码里面打印了这个值，可以看到是 0 跟 1，实际上，这是两个常量 `AVMEDIA_TYPE_VIDEO` ，`AVMEDIA_TYPE_AUDIO` 的值，如下：

![avpacket-1-3](avpacket\avpacket-1-3.png)

因此，codec_type 等于 0 代表这是一个视频流，1 代表这是一个音频流。因此咱们打印的这个包是视频包

------

然后 `AVPacket` 里面还有一个 `duration` 字段，代表这帧数据要播放 多久。时间基是 11988 ，时间基就是 1秒钟分成11988份，然后这个视频包占了 500 份，也就是当前帧要播放 0.04 秒。因此这个mp4 文件大概猜测，就是1秒钟播放24个视频包。

------

然后 `AVPacket` 里面还有一个 `pos` 字段，这个字段就代表，`data` 里面的数据是从 mp4 文件的哪个位置读取出来的，我的示例程序打印了 `data` 前面的 7个字节的数据 `0 0 2 37 6 5 ff`。

`pos` 等于 48，也就是 0x30，我们可以用 HxD 来查看一下 `juren-30s.mp4` 文件，验证一下，如下：

![avpacket-1-4](avpacket\avpacket-1-4.png)

可以看到数据是完全一致的。

---

 `AVPacket` 里面 `size` 字段代表这个视频包有多大，也就是 `data` 的长度，本文是 10201 字节大小。所以在 `juren-30s.mp4` 文件里，0x30 ~ 0x30+10201 的位置就是这个 `AVPacket` 的 `data` 数据。

------

最后，本文给出了循环读多个 `AVPacket` 的示例，读者只需把 type 变量设置成 2 即可，如下：

![avpacket-1-5](avpacket\avpacket-1-5.png)



相关阅读：

1，[《printf 格式化输出符号详细说明》](https://blog.csdn.net/xiexievv/article/details/6831194)

