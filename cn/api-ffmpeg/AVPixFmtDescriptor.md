# AVPixFmtDescriptor结构—FFmpeg数据结构详解

<div id="meta-description---">`AVPixFmtDescriptor` 是 FFmpeg 里面描述像素格式具体信息的一个结构，这个结构其实非常**常用**，像素怎么存储，排列的，都描述在这个 `AVPixFmtDescriptor`  结构里面。</div>

`AVPixFmtDescriptor` 是 FFmpeg 里面描述像素格式具体信息的一个结构，这个结构其实非常**常用**，像素怎么存储，排列的，都描述在这个 `AVPixFmtDescriptor`  结构里面。

下面来详细讲解一下这个结构体，定义在 `pixdesc.h` 文件里面。

```
typedef struct AVPixFmtDescriptor {
    const char *name;
    uint8_t nb_components; 
    uint8_t log2_chroma_w;
    uint8_t log2_chroma_h;
    uint64_t flags;
    AVComponentDescriptor comp[4];
    const char *alias;
} AVPixFmtDescriptor;
```

`AVPixFmtDescriptor` 通常是通过 `av_pix_fmt_desc_get()` 函数获取到的，接受的参数是 枚举 `AVPixelFormat`。

在  `pixdesc.c`  里面可以看到 `AVPixFmtDescriptor` 的初始化，如下：

![1-1](AVPixFmtDescriptor\1-1.png)

------

现在来逐个字段解释一下 `AVPixFmtDescriptor` 这个结构。

**1，**`const char *name`，这个很明显就是名称。

**2，**`uint8_t nb_components;` 这个字段代表 一个像素由多少个 部分组成，取值范围 1 - 4。像素通常是由 YUV 或者 RGB 组成的，后面可能会有个透明度，所以这个字段通常是 3 或者 4 。一个像素通常由 3 部分，或者 4 部分组成。有些像素格式只有 2 部分，例如 `ya16le` 

**3，**`uint8_t log2_chroma_w;`，在**水平方向**上，亮度（`luma`）样本数右移多少位 能得到 色度（`chroma`）样本数，**右移之后的值要向上取整**。

**4，**`uint8_t log2_chroma_h;`，在**垂直方向**上，亮度（`luma`）样本数右移多少位 能得到 色度（`chroma`）样本数，**右移之后的值要向上取整**。

`log2_chroma_w` 跟 `log2_chroma_h` 其实不太容易理解，在 YUV420 里面，这两个值都是 1，也就是无论在垂直角度，还是水平角度，色度样本数 都是 亮度样本数的 **2 分之一**。用下面的图来比较容易理解。

![1-2](AVPixFmtDescriptor\1-2.jpg)

X 代表亮度样本，O 代表 色度样本，无论从垂直角度还是水平角度，都是 2 分之一。

如果是 YUV422 格式，那 `log2_chroma_w` 就是 1，而 `log2_chroma_h` 是 0 。这两个值不用特别纠结，简单理解一下就行。

**5，**`uint64_t flags;`，像素标记组合，像素标记有 9 种，如下：

- `AV_PIX_FMT_FLAG_BE`，代表像素是不是大端存储。
- `AV_PIX_FMT_FLAG_PAL`，代表在 `data[1]` 里面是否有调色板。为什么需要调色板请看[《RGB色彩空间》](https://ffmpeg.xianwaizhiyin.net/base-knowledge/raw-rgb.html)
- `AV_PIX_FMT_FLAG_BITSTREAM`，比特流格式，实际上就是 `packetd` 格式，`packetd` 就是比特流， 可以简单理解为 YUV 在一个数组里面，不分开多个数组存放。比特流就是连续的数据，所以不能用多个数组切开。
- `AV_PIX_FMT_FLAG_HWACCEL`，硬件加速用的标记。
- `AV_PIX_FMT_FLAG_PLANAR`，平面格式，可以简单理解为  `YUV` 分开 3 个数组存放。
- `AV_PIX_FMT_FLAG_RGB`，像素格式是 RGB 类型的。
- `AV_PIX_FMT_FLAG_ALPHA`，像素里面有透明度。
- `AV_PIX_FMT_FLAG_BAYER`，不清楚，直译 好像是 "贝尔模板"
- `AV_PIX_FMT_FLAG_FLOAT`，像素值是浮点型。

**扩展知识：可以看到 `AVPixFmtDescriptor` 的 flags 字段才是真正决定 像素格式 是 `packetd` 还是 `planner` 格式的**。虽然命名习惯是 后面带 p 的是 `planner`，不带 p 的是 `packetd`。例如 **yuv420p** 的 `flags` 就是 `AV_PIX_FMT_FLAG_PLANAR`，如下：

![1-3](AVPixFmtDescriptor\1-3.png)

------

**6，**`AVComponentDescriptor comp[4];`，这个是 `AVPixFmtDescriptor` 里面非常重要的一个字段，定义如下：

```
typedef struct AVComponentDescriptor {
    int plane;
    int step;
    int offset;
    int shift;
    int depth;
} AVComponentDescriptor;
```

我们需要通过一个实际的例子来理解 `comp[4]`，就以 `AV_PIX_FMT_YUV420P` 跟 `AV_PIX_FMT_YUYV422` 为例，如下：

![1-4](AVPixFmtDescriptor\1-4.png)

首先分析 `AVComponentDescriptor` 里面的 `plane` 字段。 `plane` 翻译成中文是 平面的 意思，**你可以理解为 行**，FFmpeg 里面有两个英文术语，`plane` 跟 `components`，这两个术语是不一样的，`components` 可以翻译为**分量**。

例如 `AV_PIX_FMT_YUV422` 这个格式有 YUV 3 个分量，但是他只有一个 plane，因为他是一个 `packetd` 的格式，从上图可以看到 `comp` 里面的 `plane` 的值全是0 ，代表 3 个分量存储在同一行里面。

而还有一些格式，例如  `AV_PIX_FMT_P010LE` 也有 YUV 3 个分量，但是只有 2 个 `planne`，他的 Y 是单独一行，但是 UV 是混在同一行里面的。如下：

![1-5](AVPixFmtDescriptor\1-5.png)

上图中，UV 的 `plane` 都是 1，所以他们是存储在一行里面的。



------

接着分析 `AVComponentDescriptor` 的 `step` 字段，`step` 代表步长，表示水平方向连续的两个的 Y 间隔多少，提醒，我这里的 Y 是泛指分量。以 `AV_PIX_FMT_YUV420P` 为例，如下：，`step` 的值都是 1，也就是相隔一个字节，因为 Y 占 8位，而且是 plane 格式。

```
//AV_PIX_FMT_YUV420P 格式，相隔一个字节
YYYYYYY
UU
VV
```

![1-6](AVPixFmtDescriptor\1-6.png)

而对于 `AV_PIX_FMT_YUYV422` ，`step` 就不是 1 了，而是 2，4，4，如下：

![1-7](AVPixFmtDescriptor\1-7.png)

这个  2，4，4 是怎么算出来的呢？因为 `AV_PIX_FMT_YUYV422` 是 packetd 的格式，他的 YUV 是混合在一个 plane 里面的，也就是混在一行里面，如下：

```
//AV_PIX_FMT_YUYV422
Y0 Cb Y1 Cr Y3 Cb Y4 Cr Y5 Cb Y6 Cr
```

读者可以自己数一下， Y 与 Y 之间是不是隔了 2个字节，而 Cb 与 Cb 之间是不是隔了 4 个字节，Cr 与 Cr 之间是不是隔了 4 个字节。

这就是 `step` 这个字段的意思。

注意，如果 是 `AV_PIX_FMT_FLAG_BITSTREAM` ，step 的单位就是 bit。要不 单位是 byte。

------

接着分析 `AVComponentDescriptor` 的 `offset` 字段，`offset` 代表偏移。表示在当前 `plane` 中，当前 `component` （分量）的第一个样本之前有多少个字节的数据。

以 `AV_PIX_FMT_YUV420P` 为例，因为他是 一个 分量就占了 一个 `plane`，所以他的 `offset` 都是 0 ，开头就是第一个样本。如下：

![1-8](AVPixFmtDescriptor\1-8.png)

如果是 `AV_PIX_FMT_YUYV422` 格式，`offset` 就分别是 0 1 3，如下：

![1-9](AVPixFmtDescriptor\1-9.png)

这个 0 1 3 又是怎么算出来的呢？还是用 下面的例子来举例：

```
//AV_PIX_FMT_YUYV422
Y0 Cb Y1 Cr Y3 Cb Y4 Cr Y5 Cb Y6 Cr
```

开头第一个就是 Y，所以 是 第一个 `offset` 是 0，Cb 在第二个字节，所以他的 `offset` 是 1，而 Cr 在第 四 个字节，所以他的 offset 是 3。

------

接着分析 `AVComponentDescriptor` 的 `shift` 字段，这是一个右移位数。表示将对应内存单元的值右移多少位可以得到实际值。我们看一下他源代码的英文注释。

```
/**
* Number of least significant bits that must be shifted away
* to get the value.
*/
int shift;
```

`least significant bits` 可以翻译为 最低有效位。`shift` 字段就是告诉你，值的最低有效位在哪里？可能读者会觉得有点废话，对于小端存储来说，最低有效位就是第一位，这还用说嘛。

确实是的，在一个值占满内存的时候，确实是第一位就是最低有效位。所以 `AV_PIX_FMT_YUYV420p` ，`AV_PIX_FMT_YUYV422` 的 shift 都是 0 ，因为他们的 Y 位深是 8位，对齐内存之后也是 8 位。

但就是有些 像素格式，例如 `AV_PIX_FMT_P010LE` 的一个 Y 样本是占 10 位，对齐内存之后，就变成 16 位，实际上只用了 10位，这10位在 16 的哪个个位置，就由 `shift` 来确定。

可以看到 `AV_PIX_FMT_P010LE` 格式下，真正的值要 右值 6 位才能得到，`shift` 全是 6。

![2-1](AVPixFmtDescriptor\2-1.png)

------

接着分析 `AVComponentDescriptor` 的 `depth` 字段，这是位深，实际上，通俗来说，就是用多少位 来表示一个 Y，或者 一个 U，或者一个 V。

`AVComponentDescriptor`  最后 3  个字段不解释，比较少用。

------

这里做个小总结，一个像素格式的各个 分量 YUV 的存储位置，是由 `flags`， `plane`， `offset`，`step`，等等好多个字段来配合工作才能确认下来的。

------

参考文章：

1，[FFmpeg libswscale源码分析3-scale滤镜源码分析](https://www.cnblogs.com/leisure_chn/p/14355017.html)

2，[色彩空间与像素格式](https://www.cnblogs.com/leisure_chn/p/10290575.html)
