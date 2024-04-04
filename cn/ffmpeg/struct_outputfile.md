# OutputFile数据结构分析—ffmpeg.c源码分析

<div id="meta-description---">struct OutputFile 是单个输出文件的管理器。xxx</div>

`struct OutputFile` 是**单个输出文件的管理器**。之前在 `parse_optgroup()` 处理好的 `OptionsContext o ` 变量，有一部分字段会赋值给 `OutputFile` 管理器，如下：

![1-1](struct_outputfile\1-1.png)

`OptionsContext o` 变量的**另一部分字段**，会在 `open_output_file()` 里面传递给 API 函数，例如：`avformat_write_header()`，或者赋值给 `OutputStream` 的一些字段。

```
ret = avformat_write_header(of->ctx, &of->opts);
```

---

`output_files` 全局变量是一个数组，里面的成员正是 `OutputFile`，所以你在二次开发 `ffmpeg.exe` 的时候，可以通过 `output_files` 全局变量获取到所有的输出文件的信息。

```
OutputFile   **output_files   = NULL;
int         nb_output_files   = 0;
```

---

我们接下来仔细学习一下 `struct OutputFile` 的结构，如下：

```
typedef struct OutputFile {
    AVFormatContext *ctx;
    AVDictionary *opts;
    int ost_index;       /* index of the first stream in output_streams */
    int64_t recording_time;  ///< desired length of the resulting file in microseconds == AV_TIME_BASE units
    int64_t start_time;      ///< start time in microseconds == AV_TIME_BASE units
    uint64_t limit_filesize; /* filesize limit expressed in bytes */

    int shortest;

    int header_written;
} OutputFile;
```

相比 `InputFile`，`OutputFile` 数据结构的字段简直太少了，读起来太爽了。

---

`struct OutputFile` 的各个字段的解析如下：

**1，**`AVFormatContext *ctx`，容器上下文，也叫容器实例。

**2，**` AVDictionary *opts`，容器格式的参数，是从 `OptionsContext` 里面 的 `OptionGroup` 的 `format_opts` 复制过来的，如下：

```
av_dict_copy(&of->opts, o->g->format_opts, 0);
```

`opts` 会传递给 `avformat_write_header()` 函数，如下：

```
ret = avformat_write_header(of->ctx, &of->opts);
```

**3，**`int ost_index`，输出文件的第一个流在 `output_streams` 数组里面的索引，`output_streams` 数组是一个全局变量，里面包含所有输出文件的所有输出流。你二次开发 `ffmpeg.exe` 的时候，可以使用 `output_streams` 数组，获取到所有的输出流。

**4，**`int64_t recording_time`，命令行选项 `-t` 的值，设置输出文件的时长，单位是微秒，具体的功能是通过 `trim` 滤镜来实现的。

**5，**`int64_t start_time`，标记输出文件的开始时间，例如一个输入文件本来是 6 分钟的，你可以用 `-ss 120` 指定 `start_time`，这样，输出文件就会裁剪成 第 2 ~ 6分钟 的视频，前面 2 分钟丢弃。

**6，**`uint64_t limit_filesize`，限制输出文件的大小，一旦达到这个大小，输出文件立即结束。

**7，**`int shortest`，命令行选项 `-shortest` 的值，当最短的输出流结束的时候，整个文件就结束了，例如一个输出文件里面有 音频流 跟 视频流，视频流 3 分钟，音频流 5 分钟。如果启用了这个选项，音频流就会被裁剪成 3 分钟。

**8，**`int header_written`，是否已经调用了 `avformat_write_header()` 函数，往输出文件写入了头部信息。





