# new_video_stream

<div id="meta-description---">xxx</div>



打开编码器，解码器都是在 init_output_stream 里面的。



```
ret = avcodec_parameters_from_context(ost->st->codecpar, ost->enc_ctx);
```







在 init_output_stream_encode 里面设置编码器的宽高。从滤镜里面取的。

```
enc_ctx->width  = av_buffersink_get_w(ost->filter->filter);
enc_ctx->height = av_buffersink_get_h(ost->filter->filter);
enc_ctx->sample_aspect_ratio = ost->st->sample_aspect_ratio 
```

![1666019421(1)](D:\0-博客\ffmpeg_principle\ffmpeg\open_output_file\1666019421(1).png)
