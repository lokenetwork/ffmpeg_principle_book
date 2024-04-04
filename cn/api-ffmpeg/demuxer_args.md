# 如何设置解复用器参数—FFmpeg API教程

<div id="meta-description---">解复用器 （demuxer）的参数 分为 通用部分 跟 私有部分。通用部分是指所有文件格式都有的属性，例如 formatprobesize 是 MP4 跟 FLV都有的属性。
而 export_all 是只有 MP4 自己才有的属性。</div>

解复用器 （demuxer）的参数 分为 **通用部分** 跟 **私有部分**。通用部分是指所有文件格式都有的属性，例如 `formatprobesize` 是 MP4 跟 FLV都有的属性。而 `export_all` 是只有 MP4 自己才有的属性。

------

通用部分的参数可以通过以下命令来查看：

```
ffmpeg.exe -h > t.txt
```

![0-1](demuxer_args\0-1.png)

私有部分的参数可以通过指定 解复用器来查看：

```
ffmpeg.exe -hide_banner 1 -h demuxer=mp4
```

![1-3](demuxer_args\1-3.png)

------

无论是通用还是私有属性，都是使用 `AVDictionary` 来设置 demuxer 的属性的，就是最后一个参数 `AVDictionary **options`，如下：

```
int avformat_open_input(AVFormatContext **ps, const char *url, ff_const59 AVInputFormat *fmt, AVDictionary **options);
```

读者可能会疑惑，设置复用器的属性，直接设置他的字段不就行了，为什么要搞一个 `AVDictionary` 出来。因为 `FFmpeg` 社区有大量的命令行用户，他们习惯通过 **字符串传参** 来 设置 复用器，解码器等等的属性。有些用户是不会写 `C/C++` 代码的。

`AVDictionary` 这个结构就是为了 **命令行的字符串传参** 这个需求而设计出来的，它的定义如下：

```
struct AVDictionary {
    int count;
    AVDictionaryEntry *elems;
};
```

```
typedef struct AVDictionaryEntry {
    char *key;
    char *value;
} AVDictionaryEntry;
```

可以看到，非常简单，`AVDictionary` 就是一个列表，里面存储着多个 `AVDictionaryEntry`，而 `AVDictionaryEntry` 是一个 key  value 的结构体。

------

下面就来演示一下如何用 `AVDictionary` 来设置 MP4 解复用器的属性。本文的代码可再 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/demuxer_args) 上下载。

![1-4](demuxer_args\1-4.png)

代码运行结果如下：

![1-5](demuxer_args\1-5.png)

可以看到，当 `AVDictionary` 里面的值被使用了，就会剔除掉。剩下的 `option` 就是用不上的，什么情况会用不上？**就是写错了名称**。例如，上面代码中的 `export_666` 就是故意写错的。

------

上面的代码有两个重点：

**第一**，我没有在代码里申请 `AVDictionary` 的内存，那他的内存是从哪里来的？

答：是从 `av_dict_set()` 函数内部申请的内存，如果你传 `NULL` 给它，`av_dict_set()` 内部就会申请一块内存，可以看到函数的第一个参数是一个**二级指针**。

![1-6](demuxer_args\1-6.png)

**在用完 `AVDictionary` 之后，需要调 `av_dict_free()` 手动释放堆内存**。

------

**第二**，如何申请一个栈内存的 `AVDictionary` ，也就是局部变量。

答：无法做到，`AVDictionary` 只能以指针指向堆内存的方式来使用，如果你想创建一个 局部变量 `AVDictionary opts` 放在栈内存里面，编译器会报 `incomplete type` 未实现类型错误。如下：

![1-1](demuxer_args\1-1.png)

这是因为 `AVDictionary` 这个类型的定义 放在 `dict.c` 文件里面了，而 这个 `.c` 文件已经被编译器编译进去 `dll`，编译器看不到这个结构体的实现了。

------

上面讲的是 `demuxer` （解复用器）的参数设置。`muxer` （复用器）的参数也是这样设置的。不过 复用器 是通过 `avio_open2()` 函数来设置，如下：

```
int avio_open2(AVIOContext **s, const char *filename, int flags,
               const AVIOInterruptCB *int_cb, AVDictionary **options)
```

最后一个参数就是 `AVDictionary **options`。可以通过以下命令 查询 复用器支持的参数。

```
ffmpeg.exe -hide_banner 1 -h muxer=mp4
```

------

我下面的截图是用 `clion` 的 [watch point](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html) 功能 追踪命令行参数 `-movflags empty_moov`   在 `ffmpeg.c` 里面怎么传进去的。

```
ffmpeg.exe -i juren-5s.mp4 -movflags empty_moov ttt.mp4
```

![1-7](demuxer_args\1-7.png)

可以看到，就是从 `avio_open2()`  函数里传进去的。
