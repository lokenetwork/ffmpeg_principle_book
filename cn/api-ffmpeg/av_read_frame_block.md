# FFmpeg非阻塞读取AVPacket—FFmpeg API教程

<div id="meta-description---">对于常规的 mp4，flv，av_read_frame 无法非阻塞读取</div>

`AVFormatContext` 结构里面的 `flags` 字段有一个属性 `AVFMT_FLAG_NONBLOCK`，当设置了这个属性的时候，`av_read_frame()` 函数就可以非阻塞读取 `AVPacket`。

默认情况下，`av_read_frame()` 函数是阻塞型读取 `AVPacket` 的，只有读到 `AVPacket`  才会返回，或者出错直接返回 `AVERROR_EOF`，阻塞型读取是不会返回 `AVERROR(EAGAIN)` 错误码的。

但是在 `ffmpeg.c` 里面，故意把 `flags` 设置成 `AVFMT_FLAG_NONBLOCK`，如下：

![1-1](av_read_frame_block\1-1.png)

我一开始看到这个属性，以为能用在 AVIO 那里，AVIO 的场景也能设置成非阻塞读取 `AVPacket`。

但是我搜索后发现，在 `mp4`，`flv` 之类的 `demuxer` 解复用器里面，根本没有用到 `AVFMT_FLAG_NONBLOCK` 属性，所以这些常规格式是无法非阻塞读取 `AVPacket` 的，**只能用阻塞读取**。

那 `AVFMT_FLAG_NONBLOCK` 属性用在哪里？我搜索了一下，发现是用在 `libavdevice` 里面的，也就是捕捉摄像头，麦克风之类的数据的时候可以非阻塞，如下：

![1-2](av_read_frame_block\1-2.png)

![1-3](av_read_frame_block\1-3.png)
