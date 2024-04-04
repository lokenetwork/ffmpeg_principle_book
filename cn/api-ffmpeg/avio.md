# FFmpeg的自定义AVIO—FFmpeg API教程

<div id="meta-description---">ffmpeg 支持从网络流 或者本地文件读取数据，然后拿去丢给解码器解码，但是有一种特殊情况，就是数据不是从网络来的，也不再本地文件里面，而是在某块内存里面的。这时候 av_read_frame() 函数怎样才能从内存把 AVPacket 读出来呢？</div>

`ffmpeg` 支持从网络流 或者本地文件读取数据，然后拿去丢给解码器解码，但是有一种特殊情况，就是数据不是从网络来的，也不在本地文件里面，而是在某块内存里面的。

这时候 `av_read_frame()` 函数怎样才能从**内存**把 `AVPacket` 读出来呢？

FFmpeg 的开发者一早就考虑到这个问题了，所以他们提供了 自定义 `AVIO` 的功能。利用这个功能，你可以自定义 `AVIO` 的输入函数，也可以自定义 `AVIO` 的输出函数。没错，输出你也可以输出到一块内存里面，而不是保存到本地文件。

`AVIOContext` 实际上就是一个中间层，负责连接 用户层的 API 函数 与 Demuxer/Muxer ，如下：

![0-2](..\demuxer\io_open\0-2.jpg)

上图中我没有把 `Muxer` 复用的图画出来，省略了。

用户层 API 会调用 `AVIOContext` 的 `read_packet` 函数从输入文件读取数据，然后放进去 缓存 `buffer` 里面，再调用 `Demuxer` 层的函数来解析 或者 拷贝 缓存 `buffer` 的数据。 

------

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avio)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

![1-1](avio\1-1.png)

[avio](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avio) 项目 是从 [output](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/output) 项目改造来的，所以一些代码是类似的。

`avio` 项目的整个流程是这样的，先用了 `readFile()` 函数读取本地的 mp4 文件到内存，来模拟内存场景。然后调 `avio_alloc_context()` 自定义输入跟 seek 函数。

重点讲一下 `avio_alloc_context()` 函数的用法，定义如下：

```
AVIOContext *avio_alloc_context(
                  unsigned char *buffer,
                  int buffer_size,
                  int write_flag,
                  void *opaque,
                  int (*read_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int (*write_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int64_t (*seek)(void *opaque, int64_t offset, int whence));
```

参数如下：

1，`unsigned char *buffer`，这是一个指针，指向一块 `av_malloc()` 申请的内存。这是我们的自定义函数跟 `FFmpeg` 的 `API` 函数沟通的桥梁。

当 `write_flag` 为 0 时，由 自定义的回调函数 向 `buffer` 填充数据，FFmpeg API 函数取走数据。

当 `write_flag` 为 1 时，由 FFmpeg API 函数 向 `buffer` 填充数据，自定义的回调函数 取走数据。

补充，我测试的时候，这里的 `buffer` 指针跟 回调函数 里面的 `buf` 指针，好像不是同一块内存，`FFmpeg` 注释说，buffer 可能会被替换成另一块内存。

2，`int buffer_size`，`buffer` 内存的大小，通常设置为 4kb 大小即可，对于一些有固定块大小的格式，例如 TS 格式，TS流的包结构是固定长度188字节的，所以你需要设置为 188 字节大小。如果这个值设置得不对，性能会下降得比较厉害，但是不会报错。

又例如，如果输入数据是 `yuv`，你最好把 `buffer_size` 设置成一帧 `yuv` 的大小，这样上层处理起来更加方便。

3，`int write_flag` ，`write_flag` 可以是 0 或者 1，作用是标记 `buffer` 内存的用途。

4，`void *opaque`，传递给我们自定义函数用的。

5，`int (*read_packet)(...)`，输入函数的指针。

6，`int (*write_packet)(...)`，输出函数的指针。

7，`int (*seek)(...)`，seek 函数的指针。

------

再来看一下我们自定义的输入函数 `read_packet()`，如下：

![1-2](avio\1-2.png)

`buf` 跟 `buf_size` 都是 `FFmpeg` 告诉我们的，告诉我们的自定义函数，要往哪里（buf）写数据，写多少（buf_size）数据。

**`read_packet()` 函数的返回值是实际写入的数据大小，如果是一些网络流，在某个时间确实没从网络读到数据，不能返回 0 ，需要阻塞等待读到数据。如果是已经到结尾了或者网络出错了，可以返回 AVERROR_EOF，代表没有更多输入了**

------

再来看一下自定义的 seek 函数 `seek_in_buffer()`，如下：

![1-3](avio\1-3.png)

这个 seek 函数是必须实现，如果不实现这个函数，直接传 `NULL` 给 `avio_alloc_context()` ，在 mp4 格式下会导致 `av_read_frame()` 读不出来 `AVPacket`。

但是在 `flv` 格式下是可以不实现 seek 函数的，需不需要实现 `seek` 是由封装格式决定的。我测试下来，发现 `yuv` 的输入源，只要指定了 `format` 格式，就不需要实现 `seek` 函数。

`seek_in_buffer` 函数的 `offset` 跟 `whence` 参数特别重要。

`whence` 代表 seek 的类型，主要有 2 个值，如下：

1，AVSEEK_SIZE，不进行 seek 操作，而是返回 视频 buffer 整体的大小。也就是文件大小。

2，SEEK_SET，要进行 seek 操作，seek 到  `offset` 参数的位置，也就是需要 seek 到 第 `offset` 个字节。需要把 `bd->ptr` 指向第 `offset` 个字节

------

后面的逻辑就是 `av_read_frame()` 从内存读到 `AVPacket` 之后，就丢给解码器，然后再重新编码，输出成另一个 mp4。

最后需要调 `avio_context_free(&avio_ctx)` 来释放这个 自定义的 `avio` 上下文。

------

在 `main.c` 里面只定义了输入函数，输出函数是在另一个文件 `main_write.c` 里面的。你只需要修改一下 `avio.pro` 文件即可调试，如下：

```
SOURCES += main.c 
改成
SOURCES += main_write.c 
```

`main_write.c` 里面的重点代码如下：

![1-4](avio\1-4.png)

`main_write.c` 的逻辑是申请了 100M 的内存，然后把 `av_interleaved_write_frame()` 输出的数据，全部保存到这 100M 内存里面，可能没用完100M内存。

上图中我新申请了一个 `avio` 上下文实例来给输出用，不能跟输入用同一个 `avio` 实例。然后封装格式指定位 `flv` 了，因为 `flv` 格式可以不实现 `seek`。

`mp4` 格式的输出对应的 `seek` 函数我也不知道怎么实现。可能跟输入一样吧，这里埋个坑，后面填。

然后直接赋值 `fmt_ctx_out->pb` 即可

```
fmt_ctx_out->pb = avio_ctx_out;
```

内存IO模式输出，是不需要调 `avio_open2()` 的了。

------

参考文章：

1，[ffmpeg AVIOContext 自定义 IO 及 seek](https://segmentfault.com/a/1190000021378256)

2，[FFmpeg内存IO模式(内存区作输入或输出)](https://www.cnblogs.com/leisure_chn/p/10318145.html) - 叶余
