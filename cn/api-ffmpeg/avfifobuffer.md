# FifoBuffer函数库详解—FFmpeg API教程

<div id="meta-description---">FFmpeg 项目里面有一个 fifo 的实现 ，fifo 的全称是 first in first out （先进先出），而且这是一个环形的buffer内存管理器，代码实现在libavutil/fifo.h ，libavutil/fifo.c 里面。</div>

**FFmpeg** 项目里面有一个 `fifo` 的实现 ，`fifo` 的全称是 first in first out （先进先出），而且这是一个环形的**buffer内存管理器**，代码实现在 `libavutil/fifo.h ` ，`libavutil/fifo.c` 里面。

`FifoBuffer` 函数库的头文件注释非常全，其实看头文件一般就能搞懂这个函数库的用法。

下面演示一下几个例子，方便读者学习 **FifoBuffer** 函数库 的使用。本文的示例代码可以在 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/fifobuffer) 下载。

![1-1](avfifobuffer\1-1.png)

上图中，主要用了几个函数。

**1，**`av_fifo_alloc()`，申请一个 `AVFifoBuffer` 实例，可以设置初始的内存容量大小，我设置为了 10个 `MyData` 大小。现在这个内存管理器可以写进去 10 个`MyData`。

**2，**`av_fifo_grow()`，扩展 `AVFifoBuffer` 内存容量的大小，我扩展了 5 个，现在可以存储 15 个 `MyData`。

**3，**`av_fifo_size()`，`AVFifoBuffer` 里面有多少数据可以读。

**4，**`av_fifo_space()`，`AVFifoBuffer` 里面还有多少内存空间可以写。

**5，**`av_fifo_generic_write()`，往 `AVFifoBuffer` 里面**写**内存数据。

**6，**`av_fifo_generic_read()`，往 `AVFifoBuffer` 里面**读**内存数据。

**7，**`av_fifo_freep()`，释放 `AVFifoBuffer` 实例。

------

`av_fifo_generic_write()` 跟`av_fifo_generic_read()` 接受的是 `void*` 指针，**所以什么数据类型都能往这个 `AVFifoBuffer` 写**。

`AVFifoBuffer` 纯粹就是一个内存管理器，往里面写内存，然后读内存，然后把内存转成相应的结构就能操作了。

------

上面的代码，运行结果如下：

![1-2](avfifobuffer\1-2.png)

可以看到，刚创建 `AVFifoBuffer `实例的时候，他的写空间有 120 字节，读空间为 0 字节，还没有数据可以读。

有趣的是 `av_fifo_grow()` 之后，写空间没有变化，可能是因为如果空间足够，这个函数库不会去扩展内存。

然后是 for 循环写入 5 个 `MyData` 之后，就有数据可以读了，写空间也变小了一半。

然后就是把 读空间里面的数据读出来，读了两个 `MyData`。`read_space` 跟 `write_space` 也会相应变化。

------

**FifoBuffer** 函数库的代码量比较少，如果有需要可以抽出来给自己的项目使用。







