# ffmpeg命令参数类型—FFmpeg基础

<div id="meta-description---">ffmpeg命令参数分为6个类型，可执行文件、输入文件参数、输入文件名、输出文件参数、输出文件名、全局参数</div>

在继续讲其他的 ffmpeg 命令行用法之前，需要先讲 一下 ffmpeg 处理命令行参数的一些逻辑。ffmpeg 命令行 其实分为 6 个部分。如下图：

![ffmpeg-cmd-type-1-1](.\ffmpeg-cmd-type\ffmpeg-cmd-type-1-1.jpg)



1，可执行文件 ，就是 ffmpeg 那个软件。

2，输入文件参数，在 **-i** 前面那些参数，全部都作用于**当前**的输入文件。

3，输入文件名， **-i** 指定输入文件。

4，输出文件参数，输出文件名 前面的那些参数 就是输出文件参数，全部都作用于**当前**的输出文件。

5，输出文件名，juren.ts 就是输出文件名，输出文件名前面的一个参数不是 `-i`，也不是以 `-` 开始的参数。

6，全局参数， -y  跟 -benchmark 这些跟输入输出文件无关的参数就是全局参数。

上面我为什么强调 **当前** ，是因为 ffmpeg 是支持多个输入文件跟多个输出文件的，多个文件可以有各自的**独立参数**。多输入文件在复杂滤镜的场景经常使用。

------

虽然ffmpeg有很多很多的命令参数，但是只要你记住 ffmpeg 的参数类型，就会很容易学会 其他 参数的使用。

另外一点，ffmpeg 的命令行参数是没有先后顺序的，我上图是比较正确的写法，先是输入，再是输出，最后是全局参数。

但是实际上，你可以把 输出 的命令参数放在前面。如下：

```
ffmpeg -c copy -t 10 -f mpegts juren.ts -ss 00:01:32 -f flv -i juren.flv -y -benchmark
```

你甚至可以把 全局参数 插在 `-i` 前面，他依然是全局参数，不会被当成是输入文件参数，如下：

```
ffmpeg -c copy -t 10 -f mpegts juren.ts -ss 00:01:32 -f flv -y -benchmark -i juren.flv 
```

------

本文使用的素材资源下载：[百度网盘](https://pan.baidu.com/s/1YataQcYEaGQLVlzb9MMzvA )，提取码：9til 



