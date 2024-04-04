# open_input_file打开输入文件—ffmpeg.c源码分析

<div id="meta-description---">open_input_file打开输入文件</div>

`open_input_file()` 是 `ffmpeg.exe` 转换器封装的 打开输入文件的函数，在 **[FFmpeg基础API](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/)** 一章中，我们已经知道打开输入文件的 API 函数是 [avformat_open_input](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/input.html)，所以 `open_input_file()` 里面肯定会调用 `avformat_open_input`。

`open_input_file()` 的流程图如下：

![0-1](open_input_file\0-1.jpg)

由于 `open_input_file()` 的流程比较长，所以拆成了两列来画，中间的两列其实是一列。

`open_input_file()` 的定义如下：

```
static int open_input_file(OptionsContext *o, const char *filename)
```

在讲解 `open_input_file()` 的逻辑之前，需要学习一个新的结构 `struct InputFile` 以及 全局变量 `input_files`。

---

`struct InputFile` 是**单个输入文件的管理器**。之前在 `parse_optgroup()` 处理好的 `OptionsContext o ` 变量，有一部分字段会赋值给 `InputFile` 管理器，如下：

![0-2](struct_inputfile\0-2.png)

`OptionsContext o` 变量的**另一部分字段**，会在 `open_input_file()` 里面传递给 API 函数，例如：`avformat_open_input()`，或者赋值给 `InputStream` 的 `decoder_opts` 字段。

---

`input_files` 全局变量是一个数组，里面的成员正是 [InputFile](https://ffmpeg.xianwaizhiyin.net/ffmpeg/struct_inputfile.html)，所以你在二次开发 `ffmpeg.exe` 的时候，可以通过 `input_files` 全局变量获取到所有的输入文件的信息。

```
InputFile   **input_files   = NULL;
int        nb_input_files   = 0;
```

`struct InputFile` 的结构分析推荐阅读《[InputFile数据结构分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/struct_inputfile.html)》

---

了解了 `struct InputFile ` 的结构，下面开始分析一下 `open_input_file()` 函数的代码，如下：

![0-7](open_input_file\0-7.png)

开头这些代码，主要是计算出 `recording_time` 的值，这是输入文件的最大处理时长。有可能 3 分钟的文件，只需要处理第2~3分钟就行了，命令行可以这样设置的。

---

然后可能会用 `av_find_input_format()` 找到指定的容器格式，如下：

```
if (o->format) {
    if (!(file_iformat = av_find_input_format(o->format))) {
        av_log(NULL, AV_LOG_FATAL, "Unknown input format: '%s'\n", o->format);
        exit_program(1);
    }
}
...省略代码...
err = avformat_open_input(&ic, filename, file_iformat, &o->g->format_opts);
```

**avformat_open_input() 函数打开一个输入文件，会优先使用 file_iformat 参数指定的容器格式，如果 file_iformat 为空，就会根据 filename 后缀猜测，如果没有后缀，就会读取一段内容进行分析出容器格式。**

---

接下来的代码逻辑就是 **别名转换**，如下：

![0-8](open_input_file\0-8.png)

细心的读者可能注意到了，为什么这里又会设置一次 `format_opts` ？之前不是在 `opt_default()` 函数设置过了吗？

是的，如果你命令行参数写的是 `-sample_rate 44100`，如下，就会直接在 `opt_default()` 那里直接把 44100 设置进去 `format_opts`

```
ffmpeg -f alsa -sample_rate 44100 -i hw:1 -t 30 out.wav
```

上面的命令是抓取麦克风的声音，设置采样率为 44100。

但是 `sample_rate` 指定采样率，还有一个别名的，可能是以往遗留的命令名称，叫 `-ar`，这个名称在组件库没有，所以需要进行别名转换，如下：

```
ffmpeg -f alsa -ar 44100 -i hw:1 -t 30 out.wav
```

`-ar` 的赋值过程，就是先解析到 `o->audio_sample_rate` ，然后在 `open_input_file` 里面设置进去 `format_opts`。

所以上图的这些，其实都是别名转换的逻辑。

提醒：上图代码的取值是 `o->nb_audio_sample_rate - 1`，取的是最后一个，**所以如果你命令行指定了多个采样率，只会有最后一个生效。**

**重点：打开输入文件的时候，这些帧率，采样率，像素格式，在一般的容器格式，是不需要指定的，例如 flv，mp4，他们本身就保存了这些信息，但是一些裸流，例如，yuv 裸流，pcm裸流，抓屏输入等等，是没有这些信息的，需要通过命令行参数来指定。**

---

接下来的代码是匹配出命令行指定的解码器，

![0-9](open_input_file\0-9.png)

以下面的命令来讲解一下上图的逻辑：

```
ffmpeg -c:v h264_mf -i juren.mp4 juren.flv
```

`MATCH_PER_TYPE_OPT()` 宏函数的代码如下：

```
#define MATCH_PER_TYPE_OPT(name, type, outvar, fmtctx, mediatype)\
{\
    int i;\
    for (i = 0; i < o->nb_ ## name; i++) {\
        char *spec = o->name[i].specifier;\
        if (!strcmp(spec, mediatype))\
            outvar = o->name[i].u.type;\
    }\
}
```

因此下面这句代码实际上就是 匹配出带 v 的解码器，带 v 的就是视频的解码器。

```
MATCH_PER_TYPE_OPT(codec_names, str,    video_codec_name, ic, "v");
```

`MATCH_PER_TYPE_OPT()` 其实还有一个兄弟函数 `MATCH_PER_STREAM_OPT()`。

`MATCH_PER_STREAM_OPT()` 不是根据标记符 "v" 找出解码器，而是根据 AVStream 找出解码器，AVStream 是视频，就找视频的解码器，AVStream 是音频，就找音频的解码器。

```
#define MATCH_PER_STREAM_OPT(name, type, outvar, fmtctx, st)
```

---

![0-9](open_input_file\2-1.png)

我也不知道为什么要给 `ic->video_codec` 进行赋值，解码器应该是 `avcodec_find_decoder` 找到指定的解码器，然后 `avcodec_open2` 打开就行了。

这里赋值 `video_codec` 字段可能是对于某些格式，如果不赋值 `AVFormatContext`  的 `video_codec` 字段，`av_read_frame` 函数就无法从容器读取到 AVPacket？

提醒：如果你的项目解码遇到问题，可以抄 `ffmpeg.exe` 这种做法，他的健壮性比较强，估计是兼容一些特殊场景的。

---

还有一个重点代码，这样可以设置 `av_read_frame()` 函数成非阻塞状态，不过默认应该是非阻塞的吧。

```
ic->flags |= AVFMT_FLAG_NONBLOCK;
```

---

接下来就可以用 `avformat_open_input` 打开解复用器了，如下：

![2-2](open_input_file\2-2.png)

比较重要的是最后的检测选项，未使用的通常是解复用器不支持这个命令行选项，所以会直接退出。`avformat_open_input()` 函数如果使用了 `format_opts` 里面的选项，就会删除，未使用的会保留。

上面把 `codec_opts` 的选项从 `format_opts` 里面删除，估计因为有些解码器的参数，跟解复用器的参数的名称是一样的，所以需要做下兼容。

---

`open_input_file()` 函数里面有很多对于解码器的设置，估计是为了应付一些没有保存编码信息的输入文件。如下：

```
/* apply forced codec ids */
for (i = 0; i < ic->nb_streams; i++)
	choose_decoder(o, ic, ic->streams[i]);
```

上面的代码没有用 `choose_decoder` 的返回值，返回值是直接丢弃的，所以只是设置了一下 `st->codecpar->codec_id` 。

---

探测的逻辑比较简单，所以不讲解，推荐阅读《[FFmpeg打开输入文件](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/input.html)》

```
if (find_stream_info) {
        AVDictionary **opts = setup_find_stream_info_opts(ic, o->g->codec_opts);
        int orig_nb_streams = ic->nb_streams;

        /* If not enough info to get the stream parameters, we decode the
           first frames to get it. (used in mpeg case for example) */
        ret = avformat_find_stream_info(ic, opts);
		...省略代码...
}
```

---

接下来就是 seek 的逻辑，命令行如果使用了 -ss 参数，就进行 seek 操作，代码如下：

![2-3](open_input_file\2-3.png)。

![2-4](open_input_file\2-4.png)

`avformat_seek_file()` 函数的使用，推荐阅读《[avformat_seek_file函数介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avformat_seek_file.html)》。

---

seek 之后，就是添加输入流（InputStream）到全局数组 `input_streams`，这个比较复杂，推荐阅读《[add_input_stream添加输入流](https://ffmpeg.xianwaizhiyin.net/ffmpeg/add_input_stream.html)》

```
/* update the current parameters so that they match the one of the input stream */
add_input_streams(o, ic);
```

---

添加完输入流之后，就会开始添加输入文件（InputFile）到 全局数组 `input_files`。

![2-5](open_input_file\2-5.png)

这里有些同学可能疑惑，为什么是添加完输入流，然后才添加输入文件，输入流不是要绑定输入文件的吗？

是需要绑定的，不过是通过 `nb_input_files` 来绑定，`nb_input_files` 一开始是 0 ，扩容 `input_files` 之后才会变成 递增成 1。

所以需要在扩容 `input_files` 之前添加输入流。

`add_input_streams()` 函数绑定输入流到输入文件的代码如下：

![2-6](open_input_file\2-6.png)

---

`open_input_file()` 函数最后的代码就是检查一下命令行指定的，但是未使用的解码器参数，如果有未使用的就会报错。代码如下：

![2-7](open_input_file\2-7.png)

上图有些代码比较奇怪，可能是因为有些解码器的参数 跟解复用器的参数的名称是一样的，所以需要做下兼容。

---

至此 `open_input_file()` 函数的源码分析完毕，做下小总结。

`open_input_file()` 主要就是使用命令行的参数打开 demuxer，用 `avformat_open_input()` 打开，打开之后如果有 seek 操作，就会先进行 seek，然后调 `add_input_streams()` 函数添加输入流。

`add_input_streams()` 内部会初始化解码器实例，并且**使用掉命令行的解码器参数**，但是 `add_input_streams()` 不会打开解码器实例。真正打开解码器是在 `init_input_stream()` 里面做的。

`open_input_file()` 最后就是，添加输入文件到全局数组 `input_files`，赋值好 `InputFile f` 的各个字段，















