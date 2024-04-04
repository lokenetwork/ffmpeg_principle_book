# FFmpeg命令行参数分析-D—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>



ffmpeg.c 可以设置 debug 编解码器。这个重点需要讲解一下。

```
            input_streams[i]->dec_ctx->debug = debug;
```



