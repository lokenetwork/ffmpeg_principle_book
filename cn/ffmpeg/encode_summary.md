# FFmpeg转换器总结—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg.exe 转换器的编码封装总结</div>

`ffmpeg.exe` 转换器的**编码封装**的总函数就是 `reap_filter()`。

`ffmpeg.exe` 转换器的**编码封装模块** （[reap_filter](https://ffmpeg.xianwaizhiyin.net/ffmpeg/reap_filters.html)）与 **解码模块**（[process_input](https://ffmpeg.xianwaizhiyin.net/ffmpeg/process_input.html)）相比，我觉得两者的复杂度不相上下。

![1-1](encode_summary\1-1.png)

**1，**解码模块，最复杂的不是解码，而是解码之后的滤镜操作（[configure_filtergraph](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)），滤镜的配置过程非常复杂。

**2，**编码封装模块，最复杂的地方是 `do_video_out()` 函数，里面有 3 块复杂逻辑，如下：

- 《[FFmpeg可变帧率转恒定帧率详解](https://ffmpeg.xianwaizhiyin.net/ffmpeg/vfr_to_cfr.html)》
- 《[FFmpeg强制关键帧分析](https://ffmpeg.xianwaizhiyin.net/ffmpeg/force_keyframe.html)》
- 《[FFmpeg是如何调整帧率的](https://ffmpeg.xianwaizhiyin.net/ffmpeg/change_framerate.html)》

---

现在是时候为 `ffmpeg.exe` 转换器的，**命令行解析模块**，**转码前的初始化**，**解码滤镜模块**，**编码封装模块**，画一个总的流程图，如下：

![1-2](encode_summary\1-2.jpg)

有些绿色的块，我是故意画多一个的，为了方便连接起来。

图片是高清图，如果缩小模糊了，建议浏览器单独打开图片查看
