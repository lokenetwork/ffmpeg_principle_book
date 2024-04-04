# FFmpeg使用debug_ts打印全过程的pts—ffmpeg.c源码分析

<div id="meta-description---">音视频开发的一个烦恼的点，是时间总是不对。这时候你可以打开 debug_ts 选项，查看 demuxer， 解码，编码，muxer 过程中的 pts 的信息，如下</div>

音视频开发的一个烦恼的点，是时间总是不对。这时候你可以打开 `debug_ts` 选项，查看 `demuxer`， 解码，编码，`muxer` 过程中的 pts 的信息，如下：

```
ffmpeg -hide_banner -an -i juren.mp4 -t 10 juren.flv -debug_ts -y
```

![1-3](debug_ts\1-3.png)

在 `ffmpeg.c` 里一共有 6 个函数打印了 `debug_pts`，如下：

![1-4](debug_ts\1-4.png)

---

#### 1，process_input

`process_input()` 函数有两处地方是打印 `debug_ts` 的。

**第一处**是刚刚从输入文件里面读取出来 `AVPacket` 的时候，如下：

![1-5](debug_ts\1-5.png)

打印出来的日志如下：

```
demuxer -> ist_index:0 type:video next_dts:41708 next_dts_time:0.041708 next_pts:0 next_pts_time:0 pkt_pts:125 pkt_pts_time:0.0417084 pkt_dts:125 pkt_dts_time:0.0417084 off:0 off_time:0
```

---

**第二处**是对 解码 `AVPacket` 之后，因为从 读取出来 到 解码这个过程中间，可能会修改 `AVPacket` 的 pts。例如 pts 产生了 `wrap`（回环），又或者是 命令行 使用了 `-itsscale` 选项对 pts 进行缩放，等等。

![1-6](debug_ts\1-6.png)

打印出来的日志如下：

```
demuxer+ffmpeg -> ist_index:0 type:video pkt_pts:125 pkt_pts_time:0.0417084 pkt_dts:125 pkt_dts_time:0.0417084 off:0 off_time:0
```

---

#### 2，decode_video

`decode_video()` 函数里面打印的是刚从解码器里解码出来的 `AVFrame` 的时间信息，如下：

![1-7](debug_ts\1-7.png)

打印出来的日志如下：

```
decoder -> ist_index:0 type:video frame_pts:125 frame_pts_time:0.0417084 best_effort_ts:125 best_effort_ts_time:0.0417084 keyframe:0 frame_type:2 time_base:1/2997
```

---

#### 3，adjust_frame_pts_to_encoder_tb

`adjust_frame_pts_to_encoder_tb()` 函数是在 `do_audio_out()` 跟 `do_video_out()` 里面执行的，它的作用是把刚刚 从滤镜 `buffersink` 里面读取出来的 `AVFrame` 的 `pts`，从 `buffersink` 的时间基转换成 编码器的时间基，如下：

`buffersink` 的时间基默认是 `1/25`。

![1-8](debug_ts\1-8.png)

然后就打印 `AVFrame` 转换之后的 pts 以及编码器的时间基，如下：

![1-9](debug_ts\1-9.png)

```
filter -> pts:9 pts_time:0.375375 exact:9.000008 time_base:125/2997
```

---

#### 4，do_video_out

`do_video_out()` 函数有三处地方是打印 `debug_ts` 的。注意，`do_video_out` 里面有帧率变换，强制 I 帧，等等逻辑，可能会修改 pts 的。

**第一处**是在把 `AVFrame` 发送给编码器编码之前，如下：

![2-1](debug_ts\2-1.png)

```
encoder <- type:video frame_pts:8 frame_pts_time:0.333667 time_base:125/2997
```

---

**第二处**是在 从编码器读取到编码后的 `AVPacket` 的时候，如下：

![2-2](debug_ts\2-2.png)

```
encoder -> type:video pkt_pts:8 pkt_pts_time:0.333667 pkt_dts:8 pkt_dts_time:0.333667
```

---

**第三处**是把 `AVPacket` 的时间基 从 编码器时间基 转换成 `muxer` 的时间基。因为刚从编码器出来的 `AVPacket` 的 pts 的 `time_base` 是编码器自己的时间基，如果你想把它放进去 `muxer`，需要转换成 `muxer` 的时间基

如下：

![2-3](debug_ts\2-3.png)

```
encoder -> type:video pkt_pts:334 pkt_pts_time:0.334 pkt_dts:334 pkt_dts_time:0.334
```

---

#### 5，do_audio_out

`do_audio_out()` 函数也是有三处地方是打印 `debug_ts` 的，跟 `do_video_out()` 函数一样，所以不讲解了。

---

#### 6，write_packet

`write_packet()` 里面打印的 `debug_ts` 是写入输出文件之前的时间信息，如下：

![2-4](debug_ts\2-4.png)

```
muxer <- type:video pkt_pts:334 pkt_pts_time:0.334 pkt_dts:334 pkt_dts_time:0.334 size:1108
```

不过 `write_packet()` 里面用的 `ost->st->time_base` 与 `do_video_out()` 的 `ost->mux_timebase`，两者的值应该是一样的。









