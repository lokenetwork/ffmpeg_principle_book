# Linux环境使用FFmpeg的API库—FFmpeg API教程

任何 C/C++ 编译出来的 静态库 或者 动态库 在 Linux 环境的使用方法都是类似的，推荐阅读以下文章：

1. [《Linux环境编译静态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-static.html)
2. [《Linux环境编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-shared.html)

然后，如何 在 Linux 环境编译出 FFmpeg 的动态库或者静态库，可以看以下文章：

1. [《用Ubuntu18与clion调试FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html)

上面的文章只是 编译出 FFmpeg 的 .a 静态库，如果需要编译动态库，直接在 configure 的时候加上 `--enable-shared` 即可

