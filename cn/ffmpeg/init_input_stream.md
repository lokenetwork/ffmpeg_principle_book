# init_input_stream初始化输入流

<div id="meta-description---">xxx</div>

在 转换编码格式，或者转换封装格式之前 就会调 init_input_stream 来初始化输入流，如下：





推荐 call Hireral 功能，讲一下那连个图标。

​    

![1-1](D:\0-博客\ffmpeg_principle\ffmpeg\init_input_stream\1-1.png)



解码器是按照质量排序的







2，get_format() ，get_format() 这是一个回调函数，在 avcodec_open2() 打开的解码器的时候会调用 get_format()，根据 get_format 的返回值决定解码器输出哪种 像素格式，一般解码器支持输出的像素格式有限，例如 h264_cuvid 只支持输出 NV12 跟 CUDA 两种像素格式。



讲一下解码器上下文的 `get_format()` 回调函数。

```
 /**
 * callback to negotiate the pixelFormat
 * @param fmt is the list of formats which are supported by the codec,
 * it is terminated by -1 as 0 is a valid format, the formats are ordered by quality.
 * The first is always the native one.
 * @note The callback may be called again immediately if initialization for
 * the selected (hardware-accelerated) pixel format failed.
 * @warning Behavior is undefined if the callback returns a value not
 * in the fmt list of formats.
 * @return the chosen format
 * - encoding: unused
 * - decoding: Set by user, if not set the native format will be chosen.
 */
 enum AVPixelFormat (*get_format)(struct AVCodecContext *s, const enum AVPixelFormat * fmt);
```



`get_buffer()` 

这两个 API 函数是不是应该放在高级 API 里面？

