# 如何遍历编码器支持的参数—FFmpeg API教程

<div id="meta-description---">FFmpeg 项目里面有众多模块，例如容器层的 muxer 跟 demuxer，编码器，解码器，滤镜，等等。这些模块都支持各种各样的参数，那一共都有哪些参数呢？</div>

FFmpeg 项目里面有众多模块，例如容器层的 `muxer` 跟 `demuxer`，编码器，解码器，滤镜，等等。这些模块都支持各种各样的参数，那一共都有哪些参数呢？

之前的文章，都是用 `help` 命令来打印参数，如下：

```
ffmpeg.exe -h encoder=libx264
```

![1-3](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/encode_args/1-3.png)

但是在做一些桌面工具的时候，需要把**模块的参数**列出来给用户点击选择，例如把编码器参数列出去给用户选择，那用哪些API 函数可以遍历模块的参数呢？这就是本文的内容。

------

**本文以编码器模块为例，讲解如何遍历它的参数。其他的模块，  `muxer` 跟 `demuxer` ，滤镜，等等，也是一样的使用方法。**

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/av_opt_next)，编译环境是 `Qt 5.15.2` 跟 `MSVC2019_64bit` 。

本次的代码非常简洁，所以可以贴出来，如下：

```
#include <stdio.h>
#include "libavformat/avformat.h"
#include "libavutil/opt.h"
int main(){
    AVCodec *encode = avcodec_find_encoder(AV_CODEC_ID_H264);
    AVCodecContext* enc_ctx = avcodec_alloc_context3(encode);
    const AVOption *opt = NULL;
    while (opt = av_opt_next(enc_ctx, opt)) {
        printf("common opt_name is %s \n", opt->name);
    }
    while (opt = av_opt_next(enc_ctx->priv_data, opt)) {
        printf("private opt_name is %s \n", opt->name);
    }
    avcodec_free_context(&enc_ctx);
    return 0;
}
```

上面的代码中，遍历了公共参数 跟 私有参数，`AVCodecContext` 的公共参数 跟 私有参数 定义如下：

![1-1](av_opt_next\1-1.png)

`av_opt_next()` 这个函数是一个迭代的函数，定义如下：

```
const AVOption *av_opt_next(const void *obj, const AVOption *last)
{
    const AVClass *class;
    if (!obj)
        return NULL;
    class = *(const AVClass**)obj;
    ....
}
```

`av_opt_next()` 会把 `obj` 转成 `AVClass` 的结构，这是 `FFmpeg` 的套路，无论是 `struct AVCodecContext`，还是 `AVCodecContext` 里面的 `priv_data` 字段，他们的第一个字段其实都是 `AVClass`，开头的内存都是 `AVClass` 的结构。

所以 `AVCodecContext` 可以强制转成 `AVClass`，`priv_data` 也可以强制转成 `AVClass`。

很多模块都是这样做的，公共的就放在 `xxxContext` 里面，私有的就放在 `priv_data` 里面

------

最后还有一个注意的地方，`av_opt_next()` 的第二个参数要填上一次读到的 `option`，如果读到结尾了 `av_opt_next()` 会返回 `NULL`。

遍历编码器参数讲解完毕，其他的 `muxer` 跟 `demuxer`，解码器，滤镜，也是一样的用法，如果想遍历公共的参数，就传 `xxxContext` 进去，如果想遍历私有参数，就传 `priv_data` 进去。













