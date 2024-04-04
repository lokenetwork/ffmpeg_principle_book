# FFmpeg的常用函数—FFmpeg API教程

<div id="meta-description---">FFmpeg的常用函数</div>

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/functions)，这个项目应该去掉，直接用 `ffmpeg.c` 里面进行调试。

------

FFmpeg 是一个 C语言的项目，C语言标准库的功能比较简陋，所以很多东西都要自己写，造轮子。FFmpeg 已经造了不少轮子，很多很好用的函数，本文就来介绍这些常用的函数。

PS：后续会不断补充相关函数，读者也可以提意见 补充相关函数的讲解。

------

#### 1，av_strdup，复制字符串函数

`av_strdup` 函数可以很方便的创建一个字符串变量，这个变量是在堆上的，需要自己释放内存。下面对比一下原生的方式跟 用 `av_strdup` 的方式。

```
  //原生方式
  name = malloc(100);
  strcpy(name,"loken-ffmpeg");
  free(name);
  --------------------
  name = av_strdup("loken-ffmpeg");
  av_freep(name);
```

可以看到，`av_strdup` 非常地简洁，而且他内部是用的 `realloc`。正式代码如下：

![1-1](functions\1-1.png)

------

#### 2，av_new_packet，初始化 AVPacket

`av_new_packet()` 函数可以给 `AVPacket` 申请 `data` 字段的内存，并且初始化一些字段，非常好用。

```
int av_new_packet(AVPacket *pkt, int size)
{
    AVBufferRef *buf = NULL;
    int ret = packet_alloc(&buf, size);
    if (ret < 0)
        return ret;

    get_packet_defaults(pkt);
    pkt->buf      = buf;
    pkt->data     = buf->data;
    pkt->size     = size;

    return 0;
}
```

------

待分析函数：

1，`av_frame_get_buffer` 跟 `av_image_fill_arrays` 

2，`avio_find_protocol_name`，

> 当你想知道一个 URL 字符串是什么协议的时候，通过 avio_find_protocol_name 接口就能得到协议的名称，例如 http、rtmp、rtsp 等。

------

