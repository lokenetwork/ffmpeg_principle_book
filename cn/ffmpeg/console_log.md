# FFmpeg的控制台日志—ffmpeg.c源码分析

<div id="meta-description---">本文主要讲解 ffmpeg 命令在控制台输出的日志的含义，以及这些日志是如何统计出来的</div>

本文主要讲解 ffmpeg 命令在控制台输出的日志的含义，以及这些日志是如何统计出来的。下图就是日志示例。

![1-1](console_log\1-1.png)

上面的这个日志是通过 `print_report()` 函数输出的。

---

#### 1，frame，已编码的视频帧数量。

`frame` 代表的是当前已经对多少帧视频进行了编码，如果是 `-c copy` 不需要编解码的情况，`frame` 代表已经写入了多少个 `AVPacket` 。

代码如下：

![1-2](console_log\1-2.png)

`ost->frame_number` 这个变量记录的就是当前已经对多少帧数据进行编码。

---

#### 2，fps，每秒处理多少帧

fps 的全称是 frame per seconds，计算方式是用 上面的 `frame` 来除以 任务运行的时间。代码如下：

```
frame_number = ost->frame_number;
fps = t > 1 ? frame_number / t : 0;
```

上面的 `t` 变量 就是 `ffmpeg` 命令运行的时长。

---

#### 3，q，编码质量

q 的全称是 quality（质量），这个质量是放在 `AVPacket` 的 `side data` 里的，所以需要用 `av_packet_get_side_data()` 来提取出来，如下：

![1-3](console_log\1-3.png)

`AV_RL32()` 宏函数的作用是 xxx，TODO：后面补充介绍

---

以上 3 个参数，frame，fps，q 都是视频的参数。

---

#### 4，size，当前输出文件的大小

`size` 记录的是第一个输出文件的大小，如果命令行指定了 多个输入文件，这里只显示 第一个文件的大小，其余输出文件的大小是不输出显示的，代码如下：

![1-4](console_log\1-4.png)

---

#### 5，time，当前输出文件的时长

`time` 代表输出文件的时长，是通过遍历所有的输出流，提取最大的那个 pts ，然后转化成时间格式的，代码如下：

![1-5](console_log\1-5.png)

![1-6](console_log\1-6.png)

提醒：`time` 不是任务运行的时间，而是当前输出文件中那个最大的 `pts`，也可以说是当前输出文件的播放时长。

---

#### 6，bitrate，码率

`bitrate` 代表输出文件码率，计算方式是用 上面的 `size` 除以 `time`。

这里需要注意，`time` 不是任务运行的时间，而是当前输出文件中那个最大的 `pts`。所以这里的码率跟 转换完成之后的输出文件的码率是接近的。

---

#### 7，speed，处理速度

`speed` 代表 `ffmpeg` 命令行的处理速度，上图中的处理速度是 5.22 倍速，也就是处理完1小时的视频，大约需要 12 分钟。

计算方式是用 上面的 `time` 除以任务的运行时间，代码如下：

```
bitrate = pts && total_size >= 0 ? total_size * 8 / (pts / 1000.0) : -1;
speed = t != 0.0 ? (double)pts / AV_TIME_BASE / t : -1;
```

上面的 `t` 变量 就是任务运行的时长。
