# FFmpeg编译教程-高级篇

在 [FFmpeg调试环境搭建](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/debug-ffmpeg.html) 一章中，介绍了各种调试 FFmpeg 的环境，也简单编译了一下 FFmpeg 的源码。

但是之前只是编译了 FFmpeg 本身的源码，实际上 FFmpeg 的扩展性是非常强的，他定义了一种通用数据来扩展 容器封装格式，编解码器，滤镜以及网络协议。

如下：

1. 如果需要用 FFmpeg 往视频里面加文字水印，就需要添加扩展库 **freetype2** 。
2. 也可以通过加入 [x264](https://github.com/mirror/x264) 扩展库来让 FFmpeg 能用上 x264 编编解码的算法。
3. 如果 需要生成 ffplay ，就需要 加入 SDL 扩展库。

本章内容主要讲解一些常见的扩展库如何编译，以及如何被 FFmpeg 引用。

会同时讲解两种编译方式 MinGW 与 MSVC。两者编译的动态库可以相互引用。



