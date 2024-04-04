# FFmpeg安装—FFmpeg基础

<div id="meta-description---">FFmpeg安装，推荐 Windows 环境使用 gyn 的安装包，Mac 环境使用 FFmpeg 官网的安装包</div>

#### windows10环境安装：

在 windows 环境，FFmpeg 有两个网址可以下载安装包。

1，[gyn FFmpeg build](https://www.gyan.dev/ffmpeg/builds/ ) 

2，[BtbN FFmpeg-Builds](https://github.com/BtbN/FFmpeg-Builds/releases)

我比较喜欢用 gyn 的安装包，gyn 的网站如下图：

![ffmpeg-install-1-1](.\ffmpeg-install\ffmpeg-install-1-1.png)

上图中有两种  压缩包，essentials_build 只有一些基本的库，full_build 是有全部的库。在这里 推荐下载 ffmpeg-4.4.1-full_build.7z ，学习 FFmpeg 的命令行使用，不需要 shared 动态库。

------

下载完，如下图：

![ffmpeg-install-1-2](.\ffmpeg-install\ffmpeg-install-1-2.png)

FFmpeg 里面有 3 个软件：

1，ffmpeg.exe ，功能强大的 处理音视频文件的软件。

2，ffplay.exe，播放器，可以播放音视频文件。

3，ffprobe.exe，查看音视频文件的属性，在调试排查问题的时候非常有用。

------

#### Mac环境安装：

直接使用 homebrew 安装，如下：

```
brew install ffmpeg
```

------

目前，FFmpeg 官网提供了Linux，Windows，Mac 的安装包，网址 https://www.ffmpeg.org/download.html 。

![ffmpeg-install-1-3](.\ffmpeg-install\ffmpeg-install-1-3.png)
