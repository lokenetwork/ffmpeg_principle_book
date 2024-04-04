# FFmpeg解码模块总结—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg解码模块总结</div>

`ffmpeg.exe` 里面解码，以及滤镜处理的相关文章都更新完毕，如下：

**1，**《[transcode_init转码前的初始化](https://ffmpeg.xianwaizhiyin.net/ffmpeg/transcode_init.html)》

简介： ffmpeg.exe 转换编码格式，或者转换封装格式之前的操作，主要是一些初始化的操作。

**2，**《[transcode_step转码总函数](https://ffmpeg.xianwaizhiyin.net/ffmpeg/transcode_step.html)》

简介： `transcode_step()` 函数是 ffmpeg.exe 转换编码格式，或者转换封装格式的总函数。`transcode_step()` 内部每次都会读取一个 `AVPacket`，发送给解码器进行解码，解码器**可能**会输出 0个 ~ N个 `AVFrame`，然后把 `AVFrame` 发送给滤镜容器，**如果**滤镜容器有数据能出来，就会接着发送给编码器进行编码，**如果**编码器有 `AVPacket` 数据可以出来，就把 `AVPacket` 写入文件保存。

**3，**《[process_input处理输入文件](https://ffmpeg.xianwaizhiyin.net/ffmpeg/process_input.html)》

简介： `process_input()` 主要是从输入文件读取一个 `AVPacket`，然后丢给 `process_input_packet()` 函数处理这个 `AVPacket`

**4，**《[process_input_packet解码封装](https://ffmpeg.xianwaizhiyin.net/ffmpeg/process_input_packet.html)》

简介： `process_input_packet()` 是对解码函数 `decode_video()` ，`decode_audio()` 的封装。

**5，**《[decode_video解码视频帧](https://ffmpeg.xianwaizhiyin.net/ffmpeg/decode_video.html)》

简介： `decode_video()` 不仅仅会对传递进来的 `AVPacket` 进行解码，如果解码出来数据，就会调 `send_frame_to_filters()` 发送给滤镜进行处理。

**6，**《[send_frame_to_filters滤镜处理](https://ffmpeg.xianwaizhiyin.net/ffmpeg/send_frame_to_filter.html)》

简介： `send_frame_to_filters() `函数主要的职责是调 `av_buffersrc_add_frame_flags()` 把 `AVFrame` 往 滤镜里发送，但是如果滤镜没打开就会用 `configure_filtergraph` 打开滤镜。

**7，**《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》

简介：当所有的 `InputFilter` 都初始化完成，`InputFilter` 里面的 format 不等于 -1 就是初始化完成，就会调 `configure_filtergraph()` 函数来配置与打开滤镜容器。

`configure_filtergraph()` 会把所有的 `InputFilter` 与 `OutputFilter` 链接在一起。

---

整体的流程图如下：

![1-1](decode_summary\1-1.jpg)

为了让流程图简洁一些，有些 `for` 或者 `while` 循环，我省略了，你看代码是可以看到循环的逻辑的。

可以看到，整个解码的流程，最后会跑到红色的地方 `av_buffersrc_add_frame_flags`。

解码出来的 `AVFrame`，最后会发送给滤镜进行处理。

下一章节，就是讲解 如何从滤镜（OutputFilter）里面读取已经处理好的 `AVFrame`，然后进行编码，再进行 `muxer` 封装写入文件。
