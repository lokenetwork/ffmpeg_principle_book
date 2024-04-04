# FFprobe基本使用—FFmpeg基础

<div id="meta-description---">FFprobe基本使用</div>

ffprobe.exe 是一个分析工具，可以执行以下命令分析一个 音视频文件 的详细信息。

```
ffprobe -hide_banner -i juren.mp4
```

![ffprobe-usage-1-1](.\ffprobe-usage\ffprobe-usage-1-1.png)

从上图中可以看出来，juren.mp4 文件有 两个流，一个是 音频流，一个是视频流。还有视频宽度，yuv 采样格式。编码相关的信息。

ffprobe 更多的命令参数，请阅读 《FFmpeg 从入门到精通》第 2.2 章节。
