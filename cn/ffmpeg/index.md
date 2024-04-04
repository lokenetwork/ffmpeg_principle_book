# ffmpeg.c源码分析-基础篇

<div id="meta-description---">ffmpeg.exe就是 FFmpeg 官方提供的转换器，这是一个命令行工具，功能非常非常强大。比较简单的功能就是转换封装格式，滤镜处理，转换编解码格式。ffmpeg.exe 实际上可以说是 官方给的 API 函数的使用示例，只要你熟悉了 ffmpeg.exe ，基本也就熟悉了 FFmpeg 项目的大部分 API 函数的使用了</div>

`ffmpeg.exe` 就是 FFmpeg 官方提供的转换器，这是一个命令行工具，功能非常非常强大。比较简单的功能就是转换封装格式，滤镜处理，转换编解码格式。

但是他的命令行还支持各种非常复杂的语法，关于 ffmpeg 转换器的命令行用法，推荐几本书，如下：

1. [FFMPEG - From Zero to Hero](https://www.amazon.com/FFMPEG-Zero-Hero-Nick-Ferrando-ebook/dp/B08Y64XG9T/)
2. [FFmpeg Basic](http://ffmpeg.tv/)
3. [FFMPEG Quick Hacks](http://www.vsubhash.in/ffmpeg-quick-hacks-book.html)
4. [FFmpeg从入门到精通](https://item.jd.com/12349436.html)

------

本章节主要分析 `ffmpeg.exe` 的代码实现，它那些强大的功能是如何实现的？

`ffmpeg.exe` 的代码实际上不算多，加起来只有1万多行代码，相关的代码文件如下：

1. `ffmpeg.c`，`ffmpeg.exe` 的主要逻辑就放在 `ffmpeg.c` 里面。
2. `ffmpeg_opt.c`，主要是负责解析跟处理命令行参数的，里面也有一些打开输出文件，打开编码器的逻辑。
3. `ffmpeg_filter.c`，主要是负责滤镜配置的逻辑，命令行的语法可以配置很复杂的滤镜逻辑的。
4. `ffmpeg_hw.c`，硬件加速相关。
5. `ffmpeg_qsv.c`，硬件加速相关，`qsv` 是 `intel` 的硬件加速。
6. `ffmpeg_videotoolbox.c`，`videotoolbox` 是 MacOS，IOS 系统上访问硬件编解码器的一个框架。

你用到的 **FFmpeg 命令行** 的所有功能，都是在上面这些代码文件里面实现的。

`ffmpeg.exe` 实际上可以说是 官方给的 API 函数的使用示例，大部分的API函数都在 `ffmpeg.exe` 有展示。

不过也有一些 API 函数的更新速度是快于  `ffmpeg.exe` 的更新速度，就是一些新的 API 函数出来了，`ffmpeg.exe` 可能还没用上。

`FFmpeg` 的开发者很多，可能某个模块 API 的维护者跟 `ffmpeg.exe` 的维护者不是同一个人。

不过只要你熟悉了 `ffmpeg.exe` ，基本也就熟悉了 `FFmpeg` 项目的大部分 API 函数的使用了。

