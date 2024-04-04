# AVOptions详解—FFmpeg数据结构详解

<div id="meta-description---">在前面的文章 《如何设置解复用器参数》跟《如何设置编码器参数》中，我们学习了如何用 AVDictionary 来设置解复用器，编码器的参数。但是还有另一种方法也可以设置各种数据结构的参数，那就是用 AVOptions 的方式。AVOptions 相关的 API 函数，都在 libavutil/opt.h 里面，本文会选取一部分 API 函数来做讲解。</div>

在前面的文章 《[如何设置解复用器参数](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/demuxer_args.html)》跟《[如何设置编码器参数](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/encode_args.html)》中，我们学习了如何用 `AVDictionary` 来设置解复用器，编码器的参数。

但是还有另一种方法也可以设置各种数据结构的参数，那就是用 `AVOptions` 的方式。

`AVOptions` 相关的 API 函数，都在 `libavutil/opt.h` 里面，本文会选取一部分 API 函数来做讲解。

------

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avoptions)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

![1-1](avoptions\1-1.png)

上图中，我设置了两个属性  `flags` 跟 `preset`，对比 《[如何设置编码器参数](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/encode_args.html)》中 `AVDictionary` 的方法，可以发现，用 `AVOptions` 会稍微麻烦一点，你需要分清楚这个属性是公共的，还是私有的，然后传递不同的指针给 `av_opt_set()` 函数。

`AVDictionary` 的方法如下：

![1-4](encode_args\1-4.png)

`AVDictionary` 可以不用管是公共还是私有，全部丢在一起就行了。

补充：`AVDictionary` 传参，内部的实现也是先设置私有属性，然后再设置公共属性。

------

`av_opt_set()` 函数一旦返回值小于 0 ，代表设置属性失败，通常是因为写错参数名称，写错参数会返回 `AVERROR_OPTION_NOT_FOUND`，这个宏的值是 `-22`

`av_opt_set()` 是当 value 是字符串的时候用的，如果 `value` 是一个数字，那可以用 `av_opt_set_int()` 来设置编码器的参数，如果 value 是浮点数，可以用 `av_opt_set_double()`。

更多设置属性相关的函数如下：

```
/**
 * @defgroup opt_set_funcs Option setting functions
 * @{
 * Those functions set the field of obj with the given name to value.
 *
 * @param[in] obj A struct whose first element is a pointer to an AVClass.
 * @param[in] name the name of the field to set
 * @param[in] val The value to set. In case of av_opt_set() if the field is not
 * of a string type, then the given string is parsed.
 * SI postfixes and some named scalars are supported.
 * If the field is of a numeric type, it has to be a numeric or named
 * scalar. Behavior with more than one scalar and +- infix operators
 * is undefined.
 * If the field is of a flags type, it has to be a sequence of numeric
 * scalars or named flags separated by '+' or '-'. Prefixing a flag
 * with '+' causes it to be set without affecting the other flags;
 * similarly, '-' unsets a flag.
 * If the field is of a dictionary type, it has to be a ':' separated list of
 * key=value parameters. Values containing ':' special characters must be
 * escaped.
 * @param search_flags flags passed to av_opt_find2. I.e. if AV_OPT_SEARCH_CHILDREN
 * is passed here, then the option may be set on a child of obj.
 *
 * @return 0 if the value has been set, or an AVERROR code in case of
 * error:
 * AVERROR_OPTION_NOT_FOUND if no matching option exists
 * AVERROR(ERANGE) if the value is out of range
 * AVERROR(EINVAL) if the value is not valid
 */
int av_opt_set         (void *obj, const char *name, const char *val, int search_flags);
int av_opt_set_int     (void *obj, const char *name, int64_t     val, int search_flags);
int av_opt_set_double  (void *obj, const char *name, double      val, int search_flags);
int av_opt_set_q       (void *obj, const char *name, AVRational  val, int search_flags);
int av_opt_set_bin     (void *obj, const char *name, const uint8_t *val, int size, int search_flags);
int av_opt_set_image_size(void *obj, const char *name, int w, int h, int search_flags);
int av_opt_set_pixel_fmt (void *obj, const char *name, enum AVPixelFormat fmt, int search_flags);
int av_opt_set_sample_fmt(void *obj, const char *name, enum AVSampleFormat fmt, int search_flags);
int av_opt_set_video_rate(void *obj, const char *name, AVRational val, int search_flags);
int av_opt_set_channel_layout(void *obj, const char *name, int64_t ch_layout, int search_flags);
```

可以看到，头文件的注释非常全，看他注释基本就知道怎么使用了。

**`av_opt_set()` 是一个通用的函数，不仅仅可以设置编码器的属性参数，也可以设置解码器，`demuxer` ，`muxer`，重采样（音频），`scale`（图片处理），等等数据结构的属性。**

------

`libavutil/opt.h` 文件里面有一个非常重要的函数，就是 `av_opt_set_defaults()`，这个函数是用来设置默认的属性的。

在 `avcodec_alloc_context3()` 函数调用的时候就会设置好编码器的默认函数，调用流程如下：

![1-2](avoptions\1-2.jpg)

------

用 `AVOptions` 的方式设置编码器参数的方法讲解完毕了。

补充一个知识点，`ffmpeg` 有非常多有用的音视频相关的函数，例如 `xxxx`，基本上你碰到一个需求，如果需要写几十行代码，那你就要想一下 `ffmpeg` 的 `libavutil` 库里面有没有类似的函数可以直接用。
