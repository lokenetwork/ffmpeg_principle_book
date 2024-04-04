# avformat_seek_file函数介绍—FFmpeg API教程

<div id="meta-description---">在做音视频数据分析的时候，经常会遇到这样的需求，每隔5分钟抽取一帧数据进行分析。在做播放器开发的时候，也会遇到这种情况，就是拖动进度条跳转到某个位置进行播放。</div>

在做音视频数据分析的时候，经常会遇到这样的需求，每隔5分钟抽取一帧数据进行分析。

在做播放器开发的时候，也会遇到这种情况，就是拖动进度条跳转到某个位置进行播放。

如果直接用 `av_read_frame()` 不断读数据，读到第 5 分钟的 `AVPacket` 才开始处理，其他读出来的 `AVPacket` 丢弃，这样做会带来非常大的磁盘IO。

其实上面两种场景，都可以用同一个函数解决，那就是 `avformat_seek_file()`，这个函数类似于 `Linux` 的 `lseek()` ，**设置文件的读取位置**。

只不过 `avformat_seek_file()` 是用于音视频文件的。

------

`avformat_seek_file()` 函数的定义如下：

```
int avformat_seek_file(AVFormatContext *s, int stream_index, int64_t min_ts, int64_t ts, int64_t max_ts, int flags);
```

参数解释如下：

**1，**`AVFormatContext *s`，已经打开的容器示例。

**2，**`int stream_index`，流索引，但是只有在 `flags` 包含 `AVSEEK_FLAG_FRAME` 的时候才是 **设置某个流的读取位置**。其他情况都只是把这个流的 time_base （时间基）作为参考。

**3，**`int64_t min_ts`，跳转到的最小的时间，但是这个变量不一定是时间单位，也有可能是字节单位，也可能是帧数单位（第几帧）。

**4，**`int64_t ts`，要跳转到的读取位置，单位同上。

**5，**`int64_t max_ts`，跳转到的最大的时间，单位同上，通常填 `INT64_MAX` 即可。

**6，**`int flags`，跳转的方式，有 4 个 `flags`，如下：

-  `AVSEEK_FLAG_BYTE`，按字节大小进行跳转。
- `AVSEEK_FLAG_FRAME`，按帧数大小进行跳转。
- `AVSEEK_FLAG_ANY`，可以跳转到**非关键帧**的读取位置，但是解码会出现马赛克。
- `AVSEEK_FLAG_BACKWARD`，往 `ts` 的后面找关键帧，默认是往 `ts` 的前面找关键帧。

------

**`avformat_seek_file()` 函数默认是把文件的读取位置，设置到离 `ts` 参数最近的关键帧的地方。**

而且默认情况，是容器里面所有流的读取位置都会被设置，包括 音频流，视频流，字幕流。

只要流的 `discard` 属性小于 `AVDISCARD_ALL` 就会被设置。

```
AVStream.discard < AVDISCARD_ALL
```

------

`min_ts` 跟 `max_ts` 变量有一些设置的技巧。

**如果是快进的时候**，`min_ts` 可以设置得比 当前位置 大一点，例如加 2。 而 `max_ts` 可以填 `INT64_MAX`

```
min_ts = 当前位置 + 2
max_ts = INT64_MAX
```

`+2` 是为了防止某些情况，`avformat_seek_file()` 会把读取位置从 ts **往后挪一点**。

**如果是后退的时候**，`min_ts` 可以填 INT64_MIN，`max_ts` 可以设置得比 当前位置 小一点，例如减 2。

```
min_ts = INT64_MIN
max_ts = 当前位置 - 2
```

`-2` 是为了防止某些情况，`avformat_seek_file()` 会把读取位置从 ts **往前挪一点**。

------

当 `flags` 为 0 的时候，默认情况，是按时间来 `seek` 的，而时间基是根据 `stream_index` 来确定的。

如果 `stream_index` 为 `-1` ，那 `ts` 的时间基就是 `AV_TIME_BASE`，

如果`stream_index` 不等于 `-1` ，那 `ts` 的时间基就是 `stream_index` 对应的流的时间基。

这种情况，`avformat_seek_file()` 会导致容器里面所有流的读取位置都发生跳转，包括音频流，视频流，字幕流。

------

当 `flags` 包含 `AVSEEK_FLAG_BYTE`，`ts` 参数就是字节大小，代表 `avformat_seek_file()` 会把读取位置设置到第几个字节。用 `av_read_frame()` 读出来的 `pkt` 里面有一个字段 `pos`，代表当前读取的字节位置。可以用`pkt->pos` 辅助设置 `ts` 参数，

`AVSEEK_FLAG_BYTE` 是否是对所有流都生效，我后面测试一下再补充。

---

当 `flags` 包含 `AVSEEK_FLAG_FRAME`，`ts` 参数就是帧数大小，代表 `avformat_seek_file()` 会把读取位置设置到第几帧。这时候 `stream_index` 可以指定只设置某个流的读取位置，如果 `stream_index` 为 `-1` ，代表设置所有的流。

---

当 `flags` 包含 `AVSEEK_FLAG_ANY`，那就代表 `seek` 可以跳转到非关键帧的位置，但是非关键帧解码会出现马赛克。**如果不设置 `AVSEEK_FLAG_ANY`， 默认是跳转到离 `ts` 最近的关键帧的位置的。**

---

当 `flags` 包含 `AVSEEK_FLAG_BACKWARD`，代表 `avformat_seek_file()`  在查找里 `ts` 最近的关键帧的时候，会往 `ts` 的后面找，默认是往 `ts` 的前面找关键帧。

提醒：`AVSEEK_FLAG_BYTE` ，`AVSEEK_FLAG_FRAME`，`AVSEEK_FLAG_ANY` 这 3 种方式，有些封装格式是不支持的。

------

下面通过一个例子来演示 `avformat_seek_file()` 函数的用法。代码下载地址：[GitHb](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avformat_seek_file)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

![1-1](avformat_seek_file\1-1.png)

![1-2](avformat_seek_file\1-2.png)

运行结果如下：

![1-3](avformat_seek_file\1-3.png)

可以看到，跳转之后，后面 `av_read_frame()` 读取到的 `AVPacket` 的 `pts` 跟 `pos` 都有很大的偏移了。

`avformat_seek_file()` 函数介绍完毕。

扩展知识：`avformat_seek_file()` 对应的旧版函数是 `av_seek_frame()`



