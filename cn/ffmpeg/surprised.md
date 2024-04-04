# FFmpeg转换器的小彩蛋—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

本文主要介绍 FFmpeg转换器 里面的一些小彩蛋，平时不太容易注意的功能，但在一些场景下还是比较有用的。

#### 1，av_pkt_dump_log2 打印 AVPacket 的信息

可以在命令行指定 `-dump` 跟 `-hex` 来打印从输入文件里面读取的每一个 `AVPacket` 的信息，如下：

```
ffmpeg -an -i juren.mp4 -vframes 1 juren.flv -dump -hex
```

由于 `-dump` 跟 `-hex`  是[全局参数](https://ffmpeg.xianwaizhiyin.net/base-ffmpeg/ffmpeg-cmd-type.html)，所有你把它们放在哪里都是可以的。

上面的命令只打印了 1个 `AVPacket`，如下：

![1-1](surprised\1-1.png)

这个功能在 `ffmpeg.c` 里面是调 `av_pkt_dump_log2()` 函数完成的，所以放我们需要打印 `AVPacket` 信息的时候，也可以用这个函数。

![1-2](surprised\1-2.png)

---

#### 2，丢弃新发现的流

在 `ffmpeg.c` 里面，如果发现一个 `AVPacket` 的流ID 比文件记录的流数量还大，就会直接丢弃这个 `AVPacket`，如下：

```
  /* the following test is needed in case new streams appear
       dynamically in stream : we ignore them */
    if (pkt->stream_index >= ifile->nb_streams) {
        report_new_stream(file_index, pkt);
        goto discard_packet;
    }
```

具体什么情况 `AVPacket` 的流ID 比文件记录的流数量还大，我也不清楚。

