# ffmpeg编码格式转换—FFmpeg基础

<div id="meta-description---">ffmpeg编码格式转换，使用 -c 选项即可实现编码转换</div>

本文主要讲解 ffmpeg 的**编码格式转换**。本文使用的素材资源下载：[百度网盘](https://pan.baidu.com/s/1YataQcYEaGQLVlzb9MMzvA )，提取码：9til 。素材文件如下：

------

我们可以通过 命令 `ffmpeg -codecs` 查看 FFmpeg 支持的编解码标准，如下：

![ffmpeg-codec-1-1](.\ffmpeg-codec\ffmpeg-codec-1-1.png)

上面的 D 代表 decodec （解码），E 代表 encodec（编码）。有些标准在 ffmpeg 里面只实现了 解码，没有实现 编码。

`ffmpeg -codecs`  输出的信息太多，我通常会重定向到 一个 文件里面查看，如下：

```
ffmpeg -codecs > codecs.txt
```

------

现在演示一下 H.264 转 H.265 的 命令行语法，由于 FLV 目前官方不支持 H265，只能通过扩展字段自己加H265，没有标准。所以本文用 MP4 格式来演示，MP4 对 H.265 的支持是有标准定义的。

```
ffmpeg -i juren.mp4 -c:v hevc -c:a copy juren-h265.mp4
```

上面的 `-c` 是指定编码格式 ，v 跟 a 分别代表 视频 跟 音频，视频以 hevc 编码，音频不变直接 copy 到输出文件即可。H.265 就是 hevc。

本文的是简单用法，实际 H.264 跟 H.265 是有非常多的参数可以选择的，推荐阅读《FFmpeg从入门到精通》第四章。

