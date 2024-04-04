# open_output_file打开输出文件—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

`open_output_file()` 打开输出文件的流程，跟 `open_input_file()` 打开输入文件的流程是非常类似的。

**都是创建一个文件管理器，输出的文件管理器是 `struct OutputFile`，然后添加输出流 `OutputStream`，创建编码器上下文 `ost->enc_ctx`。**

不过跟打开输入文件一样，都只是创建了编码器实例，但是都**还没真正打开编码器**。

打开编码器是在 `init_output_stream()` 函数里面的，如下：

```
if ((ret = avcodec_open2(ost->enc_ctx, codec, &ost->encoder_opts)) < 0) {...}
```

打开解码器是在 `init_input_stream()` 函数里面的，如下：

```
if ((ret = avcodec_open2(ist->dec_ctx, codec, &ist->decoder_opts)) < 0) {...}
```

----

在讲解 `open_output_file()` 函数的逻辑之前，需要先学习 `struct OutputFile ` 结构，推荐阅读《[OutputFile数据结构分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/struct_outputfile.html)》

---

`open_output_file()` 函数的流程图如下：

![0-1](open_output_file\0-1.jpg)

由于 `open_output_file()` 的流程比较长，所以拆成了两列来画，中间的两列其实是一列。

`open_output_file()` 函数的逻辑其实比上面的流程图更加复杂的，有比较多的细枝末节的逻辑，例如一些赋值操作，`nb_stream_maps` 的逻辑，处理 metadata，chapters，programs 的数据等等，这些逻辑其实在简单场景下不会跑进去，所以可以先不管。

我说的**简单场景**，是指下面这样一条命令。`juren-5s.mp4` 的下载地地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/ffplay/juren-5s.mp4)

```
ffmpeg -i juren-5s.mp4 juren-5s-copy.mp4
```

本章节，大部分的代码分析都是基于简单场景的。

------

简单场景下，没有使用复杂滤镜的选项 `-filter_complex`，所以只会调 `init_simple_filtergraph()` 函数来初始化滤镜。

`ffmpeg.exe` 的转换器就是这么实现的，即便你命令行没有使用滤镜，他也会创建一个空白的滤镜，这是为了让逻辑更加通用。

`open_output_file()` 函数里面有比较多的复杂滤镜的逻辑，通常都是判断 `nb_filtergraphs` 是否大于 0，这些逻辑你可以暂时不看。

简单场景下，`nb_filtergraphs` 会是 0 。

---

`open_output_file()` 函数里面有 4 个重点的地方。

**第一个重点：**选出最高分辨率的视频流，选出最多声道数的音频流。

如果 mp4 文件有多个视频流，多个音频，`ffmpeg.exe` 转换器会选出最好的那个来进行处理，如下：

![0-1-1](open_output_file\0-1-1.png)

---

**第二个重点：**`new_video_stream()` 函数的 最后一个参数，如下：

![0-2](open_output_file\0-2.png)

最后一个参数 `source_index` 代表输出流对应的输入流，在简单场景下，输出流都是对应一个输入流。

但是在复杂滤镜下，有可能是多个输入流合并输出一个输出流，所以在复杂滤镜下，`source_index` 会设置成 -1，代表没有对应的输入流。

---

**第三个重点**：[初始化简单滤镜](https://ffmpeg.xianwaizhiyin.net/ffmpeg/init_simple_filtergraph.html)，如下：

![0-3](open_output_file\0-3.png)

注意 `ist->decoding_needed` 会因此变成 非 0，所以对应的输入流会进行解码操作。

---

**第四个重点：**设置出口滤镜的宽高等等信息，由于出口滤镜出来的 `AVFrame` 会进行编码，然后保存进去容器，所以出口滤镜的宽高，采样等等，必须更容器的保持一致。

![0-4](open_output_file\0-4.png)

---

后面的都是一些简单场景不会跑进去的逻辑，如下：

![0-5](open_output_file\0-5.png)

至此，`open_output_file()` 函数的源码分析完毕。

