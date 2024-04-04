# FFmpeg输出多文件—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg输出多文件的实现原理</div>

虽然之前在讲解 FFmpeg原理 的时候，都只指定了一个输出文件，但实际上 `ffmpeg` 命令是支持多个输出文件的，如下：

```
ffmpeg -an -i juren.mp4 juren.ts juren.flv
```

上面这条命令，输出文件有两个，一个是 `juren.ts`，一个是 `juren.flv`。

你可以在此基础上加一些复杂的滤镜操作，然后把滤镜的输出给多个输出文件，如下：

```
ffmpeg -an -i juren.mp4 -complex_filte xxx juren.ts juren.flv
```

---

`ffmpeg` 之所以能输出多个文件，得益于它内部良好的数据结构的设计。之前说过，即便你命令行没有使用 滤镜，他也会插入一个 null 的空滤镜，所以下面两条命令是等价的。

```
ffmpeg -an -i juren.mp4 juren.ts juren.flv
```

```
ffmpeg -an -i juren.mp4 -vf "null" juren.ts juren.flv
```

下面分析一下上面的命令在 `ffmpeg.c` 实现原理。

首先，由于有 两个输出文件，所以 `output_file[]` 数组里有两个元素， `output_stream[]` 数组里也有两个元素，如下：

![1-2](many_output_file\1-2.png)

---

`ffmpeg.c` 是怎么实现多个输出文件的呢？

实际上是这样的，他创建了两个 `FilterGraph` ，这样，解码出来 `AVFrame` 之后，就会往 两个 `FilterGraph` 中的 `InputFIlter` 发送，然后再从 `OutFIlter` 读取出来 `AVFrame`，往编码器发送。

![1-4](many_output_file\1-4.png)

![1-3](many_output_file\1-3.png)

各个数据结构之间的关系可以参考《[init_simple_filtergraph初始化简单滤镜](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph.html)》的图片，如下：

![0-3](init_simple_filtergraph\0-3.jpg)



