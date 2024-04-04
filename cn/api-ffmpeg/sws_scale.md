# sws_scale图像缩放函数介绍—FFmpeg API教程

<div id="meta-description---">sws_scale()是libswscale库里面一个非常常用的函数，可以对图像进行缩放，或者转换格式跟色彩空间</div>

`sws_scale()` 是 `libswscale` 库里面一个非常常用的函数，它的功能如下：

**1，**对图像的大小进行缩放。

**2，**转换图像格式跟颜色空间，例如把 `YUYV422` 转成 `RGB24` 。

**3，**转换像素格式的存储布局，例如把 `YUYV422` 转成 `YUV420P` ，`YUYV422` 是 `packed` 的布局，YUV 3 个分量是一起存储在 `data[0]` 里面的。而 `YUV420P` 是 `planner` 的布局，YUV 分别存储在 `data[0]` ~ `data[2]`。

`sws_scale()` 转换到不同的颜色空间的时候，例如 `yuv` 转 `rgb`，或者 `rgb` 转 `yuv`，**通常是有损失的**，推荐阅读《[RGB与YUV相互转换](https://ffmpeg.xianwaizhiyin.net/base-knowledge/raw-yuv-to-rgb.html)》

------

下面来演示一下 `sws_scale()` 函数的用法，本文的代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/sws_scale)，编译环境是 `Qt 5.15.2` 跟 `MSVC2019_64bit` 。

![0-0](sws_scale\0-0.png)

先讲解 `sws_scale_1` 项目的重点代码，如下：

```
int sws_flags = SWS_BICUBIC;
AVFrame* result_frame = av_frame_alloc();
//定义 AVFrame 的格式，宽高。
result_frame->format = AV_PIX_FMT_BGRA;
result_frame->width = 200;
result_frame->height = 100;
```

`sws_scale_1` 项目一开始的时候，就创建了一个 `result_frame` 变量，用来保存转换之后的图像内容，注意，这个 `av_frame_alloc()` 函数只会申请了 `AVFrame` 这个结构体的内存，`AVFrame` 里面的 `data buffers` 内存是没有申请的。

所以我们需要指定一下 `AVFrame` 的像素格式，宽高之后，再调 `av_frame_get_buffer()` 函数来申请 buffers 的内存。如下：

```
ret = av_frame_get_buffer(result_frame, 1);
```

**重点：你必须指定像素格式，宽高，它才知道要申请多少内存**。最后一个参数是对齐参数，用的是 1 字节对齐，推荐阅读《[FFmpeg内存对齐](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/mem_align.html)》

------

申请完 `result_frame` 变量的内存，就可以开始进行 `sws_scale()`  转换了，如下：

![0-1](sws_scale\0-1.png)

上图中的 `img_convert_ctx` 变量是一个 `sws` 实例（上下文）。

`sws_getCachedContext()` 函数的定义如下：

```
struct SwsContext *sws_getCachedContext(struct SwsContext *context,
                                        int srcW, int srcH, enum AVPixelFormat srcFormat,
                                        int dstW, int dstH, enum AVPixelFormat dstFormat,
                                        int flags, SwsFilter *srcFilter,
                                        SwsFilter *dstFilter, const double *param);
```

`sws_getCachedContext()` 函数的名字之所以带 **Cached**，是因为如果 `context` 是 NULL，`sws_getCachedContext()` 函数内部就会申请 `sws` 实例的内存。如果 `context` 已经有内存了，就会复用，不会重新申请内存。

第 2 ~ 8 个参数就是 原始图像的信息 跟 目标图像的信息，宽高，像素格式。

`flags` 参数是指使用**哪种转换算法**，一共有很多种算法，定义在 `libswscale/swscale.h`

```
/* values for the flags, the stuff on the command line is different */
#define SWS_FAST_BILINEAR     1
#define SWS_BILINEAR          2
#define SWS_BICUBIC           4
#define SWS_X                 8
#define SWS_POINT          0x10
#define SWS_AREA           0x20
#define SWS_BICUBLIN       0x40
.....
```

不过比较常用的算法就是 `SWS_BICUBIC`，`ffplay` 播放器也是用的这个算法。

`sws_getCachedContext()` 函数最后面 3 个参数很少用，直接填 NULL 就行。

------

`sws_scale()` 函数的定义如下：

```
int sws_scale(struct SwsContext *c, const uint8_t *const srcSlice[],
              const int srcStride[], int srcSliceY, int srcSliceH,
              uint8_t *const dst[], const int dstStride[]);
```

比较重要的是 `srcSliceY` 参数，这个**应该是**像素内存的偏移位，偏移多少才是真正的像素数据，本文没有偏移，所以填 0 即可。

------

转换成功之后，就可以把 `BGRA` 图像保存进去文件验证转换的结果，如下：

```
int save_rgb_to_file(AVFrame *frame, int num){
    //拼接文件名
    char pic_name[200] = {0};
    sprintf(pic_name,"./rgba_8888_%d.yuv",num);

    //写入文件
    FILE *fp = NULL;
    fp = fopen(pic_name, "wb+");
    fwrite(frame->data[0] , 1, frame->linesize[0] * frame->height, fp);
    fclose(fp);
    return 0;
}
```

由于 `AV_PIX_FMT_BGRA` 格式是 `packetd` 布局的，所以只有 `data[0]` 有数据，直接把 data[0] 写进去文件即可，

最后用 **7yuv** 软件打开查看内容，如下：

![0-2](sws_scale\0-2.png)

可以看到，确实缩小到了 **200 x 100** 的宽高，像素格式也是 **RGBA8888** ，是正确的。

------

如果需要进行编码，把转换后的 `AVFrame` 丢给 编码器即可，但是前提是需要设置好 PTS 。

但是有些场景，是不需要编码的，不需要编码，就不需要申请 `AVFrame`。有些场景只需要 `sws_scale()` 转换之后的内存，把内存丢给播放器播放，丢给网络，或者丢给另一个系统处理。

下面就介绍一下另一种申请图像内存的方式，流程如下：

```
//根据像素格式，宽高，确定内存的大小。
int buf_size = av_image_get_buffer_size(result_format, result_width, result_height, 1);
//申请内存
uint8_t* buffer = (uint8_t *)av_malloc(buf_size);
//把内存 映射 到数组，因为有些像素布局可能是 planner
av_image_fill_arrays(pixels, pitch, buffer, result_format, result_width, result_height, 1);
```

示例代码在 **sws_scale_2** 项目里面，重点如下：

![0-4](sws_scale\0-4.png)

![0-5](sws_scale\0-5.png)

最后运行之后的结果如下，确实转换成了 **300 x 300** 的图像。

![0-3](sws_scale\0-3.png)

------

其实还有一种更简单的申请图像内存的方法，可以把 确定内存大小  ➔ 申请内存  ➔ 把内存映射到数组，这 3 步合成一步。

这就是 `av_image_alloc()` 函数，定义如下：

```
/**
 * Allocate an image with size w and h and pixel format pix_fmt, and
 * fill pointers and linesizes accordingly.
 * The allocated image buffer has to be freed by using
 * av_freep(&pointers[0]).
 *
 * @param align the value to use for buffer size alignment
 * @return the size in bytes required for the image buffer, a negative
 * error code in case of failure
 */
int av_image_alloc(uint8_t *pointers[4], int linesizes[4],
                   int w, int h, enum AVPixelFormat pix_fmt, int align);
```

`av_image_alloc()` 函数的用法，就不演示了，读者自行探索。

------

至此，`sws_scale`图像缩放函数 就介绍完毕了。

如果觉得 `sws_scale()` 函数使用起来比较麻烦，可以使用 [scale滤镜](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/scale.html) 来实现转换的需求。 `scale` 滤镜也支持各种转换算法。

补充一个知识点：图像相关的API函数在 `libavutil/imgutil.h` 能找到，`AVFrame` 相关的函数在 `libavutil/frame.h`  里面。`AVFrame` 是一个管理图像数据的结构体，`AVFrame` 里面有指针指向真正的图像内存数据。







