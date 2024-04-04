# FFmpeg支持哪些封装格式—FFmpeg API教程

<div id="meta-description---">我们在开发一些桌面转码软件的时候，需要把所有的封装格式列出来给用户点击选择，这时候应该调哪些API函数能获取大封装格式列表呢？这就是本文的内容。</div>

我们可以通过下面的命令查询 FFmpeg 支持哪些封装格式，如下：

```
ffmpeg.exe -hide_banner 1 -muxers
```

![1-1](av_demuxer_iterate\1-1.png)

虽然命令行可以打印这些信息，但是我们在开发一些桌面转码软件的时候，需要把所有的封装格式列出来给用户**点击选择**，这时候应该调哪些API函数能获取大封装格式列表呢？这就是本文的内容。

------

本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/av_demuxer_iterate)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

```
#include <stdio.h>
#include "libavformat/avformat.h"

int main(){
    void* ifmt_opaque = NULL;
    const AVInputFormat *ifmt  = NULL;
    while ((ifmt = av_demuxer_iterate(&ifmt_opaque))) {
        printf("ifmt name is %s \n",ifmt->name);
    }

    void* ofmt_opaque = NULL;
    const AVOutputFormat *ofmt  = NULL;
    while ((ofmt = av_muxer_iterate(&ofmt_opaque))) {
        printf("ofmt name is %s \n",ofmt->name);
    }
    return 0;
}
```

用 `av_demuxer_iterate()` 与 `av_muxer_iterate()` 这两个函数，即可遍历所有的封装格式，

注意 `ifmt_opaque` 跟 `ofmt_opaque`，这两个变量，**这两个变量保存的是当前的迭代状态**，刚开始遍历的时候需要传 NULL，他内部会改变这个指针的指向，指向当前的迭代状态。

简单来说，就是需要定义一个 `void*` 的 NULL 指针，传进去就行了。

运行结果如下：

![1-2](av_demuxer_iterate\1-2.png)

扩展知识：如果需要遍历封装格式里面的参数，使用 `av_opt_next()` 即可，推荐阅读《[如何遍历编码器支持的参数](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/av_opt_next.html)》



