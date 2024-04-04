# FFmpeg转换封装格式—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg转换封装格式的代码分析</div>

FFmpeg 转换封装格式的命令是这样的，如下：

```
ffmpeg -i juren-30s.mp4 -c copy juren-30.flv
```

上面的命令把 原来的 `mp4` 封装格式，转成了 `flv` 格式。一定要 加 `-c copy` 选项，如果不加，即便转换后的格式也是 mp4 ，也会进行编解码。

```
ffmpeg -i juren-30s.mp4 juren-30-2.mp4
```

上面的命令中，虽然输入输出文件都是 mp4 格式，但是内部是会进行编解码操作的。

------

本章节，就是分析 `ffmpeg.exe` 里面的转换封装格式的逻辑的，以下面这条命令来做讲解。

```
ffmpeg -i juren-30s.mp4 -c copy juren-30.flv
```

`juren-30s.mp4` 可在 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/discard_audio/juren-30s.mp4) 下载，`ffmpeg` 命令行的调试环境搭建，推荐阅读《[用Ubuntu18与clion调试FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html)》

------

本章节的目录如下：

1. 《命令行参数解析》
2. 《xx》
3. 《xxx》
4. 《xxxx》
