# FFmpeg命令行参数分析-vn—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg 是如何去掉视频流的？-vn 写在输入文件前面 跟 写在 输出文件前面有什么区别？
</div>

`vn` 的全称是 video no，也就是去掉视频的意思。这也是一个可以用于 输入 或者 输出的 参数，定义如下：

```
{ "vn", OPT_VIDEO | OPT_BOOL  | OPT_OFFSET | OPT_INPUT | OPT_OUTPUT,{ .off = OFFSET(video_disable) }, "disable video" }
```

---

#### vn 参数作用于输出文件

`vn` 一个比较常用的场景就是在输出的时候，把视频流删除，只保留音频流，命令如下：

```
ffmpeg -i juren.mp4 -vn juren.flv
```

`-vn` 去掉视频的实现原理如下：

**1，**`vn` 参数会赋值到 `OptionsContext` 的 `video_disable` 字段。常规操作

**2，**`video_disable` 会在 `open_output_file()` 函数被使用，导致 `new_video_stream()` 函数没有执行，所以 `muxer` 里没有视频的流，如下：

![1-1](cmd_arg_vn\1-1.png)

虽然输出文件没有了视频流，但是输入文件还是会把视频流的 `AVPacket` 读取出来，那读取出来的 `AVPacket` 会如何处理呢？

实际上是这样的，在 `ffmpeg.c` 里面，`InputStream` 是管理输入流的数据结构，默认所有的 `InputStream` 的 `discard` 字段都是 1，默认是丢弃的，所以读取出来的 `AVPacket` 默认就是会丢弃，代码如下：

![1-2](cmd_arg_vn\1-2.png)

![1-3](cmd_arg_vn\1-3.png)

---

那什么情况下 `InputStream` 的 `discard` 字段会被设置为 0 呢？有两种情况。

**1，`new_video_stream()` 函数被执行的情况，如下：**

![1-4](cmd_arg_vn\1-4.png)

提示：`new_video_stream` 里面会执行 `new_output_stream`

由于本文的命令使用了 `-vn` 选项，所以 `new_video_stream()` 函数不会被执行，所以输入文件的视频流 `AVPacket` 默认就是丢弃的。

---

**2，在复杂滤镜里面使用了 输入文件的视频流，命令如下：**

```
ffmpeg -i juren.mp4 -complex_filter "[0:v]...省略" juren.flv
```

上面的命令使用了 `[0:v]`，也就是第一个输入文件的视频流，所以 `discard` 字段会被设置为 0 ，代码如下：

![1-5](cmd_arg_vn\1-5.png)

---

#### vn 参数作用于输入文件

`vn` 用在输入文件前面的场景，我个人来说是比较少用的，不过是支持这样设置。命令如下：

```
ffmpeg -vn -i juren.mp4 juren.flv
```

上面这条命令的作用是 ，在 `ffmpeg.c` 里面的实现流程如下：

**1，**`vn` 参数会赋值到 `OptionsContext` 的 `video_disable` 字段。常规操作

**2，**`video_disable` 变量转移给 `ist->user_set_discard`，代码如下：

![1-6](cmd_arg_vn\1-6.png)

那 `ist->user_set_discard` 在哪里被使用了呢？在 `new_output_stream()` 里面，如下：

![1-7](cmd_arg_vn\1-7.png)

上图中是直接设置的 `AVStream` 里面的 `discard` 字段，一旦 `AVStream` 里面的 `discard` 字段设置为非 0 ，`av_read_frame()` 读取输入文件的时候，就不会返回这个流的 `AVPacket`。

因此在输入文件前面加 `-vn`，可能会带来一些性能提升，因为 `av_read_frame()` 就不会读取出来视频流的 `AVPacket`。

而如果在输出文件前面加 `-vn`， `av_read_frame()` 会把 `AVPacket` 读取出来然后再丢弃。会多一些内存释放的操作。
