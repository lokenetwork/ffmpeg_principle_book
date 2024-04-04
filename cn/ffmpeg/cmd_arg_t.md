# FFmpeg命令行参数分析-t—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg 通过 -t 限制时长的实现原理</div>

`t` 参数的全称是 recording_time，用于限制 输入源 或者 输出源 的时间，定义如下：

![1-1](cmd_arg_t\1-1.png)

用法如下：

```
ffmpeg.exe -i juren.mp4 -t 10 juren.flv
```

```
ffmpeg.exe -t 10 -i juren.mp4 juren.flv
```

---

#### t 参数作用于输出源

当 `t` 参数作用于输出源的时候，这个参数就会被赋值给 `OutputFile` 的 `recording_time` 字段，如下：

```
# open_output_file 函数的代码
of->recording_time = o->recording_time;
```

然后这个 `of->recording_time` 变量是怎么用来限制 输出文件的时长的内容呢？

答：在 `configure_output_video_filter()` 与`configure_output_audio_filter()` 函数里面，如下：

![1-1-2](cmd_arg_t\1-1-2.png)

![1-1-3](cmd_arg_t\1-1-3.png)

从上图可以看到，在 **输出** 的滤镜链条里面插入了 `trim` 滤镜，这样当 `buffersink` 滤镜输出了 10 秒的数据，`avfilter_graph_request_oldest()` 函数就会自动返回 `AVERROR_EOF` 错误码，这样 `ffmpeg` 命令行就会结束了。

---

#### t 参数作用于输入源

当 `t` 参数作用于输出源的时候，这个参数都会被赋值给 `InputFile` 的 `recording_time` 字段，如下：

```
# open_input_file 函数的代码
f->recording_time = o->recording_time;
```

跟输出限制时长的情况类似，`f->recording_time` 也是被用在 `trim` 滤镜里面的，不过是在 `configure_input_video_filter()` 与`configure_input_audio_filter()` 函数里面，如下：

![1-1-4](cmd_arg_t\1-1-4.png)

从上图可以看到，在 **输入** 的滤镜链条里面插入了 `trim` 滤镜，这样当往 `buffer` 滤镜输入了 10 秒的数据，`avfilter_graph_request_oldest()` 函数就会自动返回 `AVERROR_EOF` 错误码，这样 `ffmpeg` 命令行就会结束了。

TODO：这种效果不是跟输出一样吗？可能在 `-t` 作用于输入的情况下， `avfilter_graph_request_oldest()` 函数不会结束，后面验证一下。

---

最后再补充一点，虽然 `check_recordind_time()` 函数也用到了 `of->recording_time` ，但是我测试发现，`check_recordind_time()` 函数并没有发挥限制时长的作用，所以 `check_recordind_time()` 函数 很可能是多余的，因为已经用 `trim` 滤镜限制了输出时长。

![1-2](cmd_arg_t\1-2.png)

![1-3](cmd_arg_t\1-3.png)



