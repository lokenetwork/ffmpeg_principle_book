# 多线程读取输入文件—ffmpeg.c源码分析

<div id="meta-description---">ffmpeg.exe 如何开启多线程读取输入源，提高处理速度</div>

`ffmpeg.exe` 转换器在默认的配置下，当有多个输入源的时候，是开启多线程从输入源里面读取 `AVPacket` 的，一个输入源对应一个线程（input_thread）。

什么是默认的配置？

在 `configure` 运行的时候，它会去检测当前系统是否能正常使用多线程的API函数，例如，`pthreads`，`w32threads`，`os2threads` 的 API 函数都会检测一下，只要能正常使用其中一种线程 API，就会默认开启 FFmpeg 的多线程功能。

提示：FFmpeg 的多线程代码是通过 HAVE_THREADS 宏来开启的，`configure` 脚本生成的 `config.h` 头文件，会定义这个 `HAVE_THREADS` 宏。

你也可以通过 `--disable-pthreads`，`--disable-w32threads`，`--disable-os2threads`，禁用掉多线程的功能，通常我是在调试学习 FFmpeg 的代码会禁止多线程，因为单线程的代码逻辑相对简单。

因为各个操作系统提供的 线程API 是不一样的，所以 FFmpeg 对其进行了封装，给外部使用提供了统一的函数方法，推荐阅读《[FFmpeg的多线程介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/thread.html)》

----

现在正式进入本文的主题，在 `ffmpeg.c` 里面是如何使用多线程去读取输入文件的，使用多线程又有什么好处？

在 `ffmpeg.c` 里面，创建多线程的流程如下：

![0-1](multi_thread\0-1.jpg)

可以看到创建线程的函数就是 `init_input_threads()`，这个函数的重点如下：

![0-2](multi_thread\0-2.png)

`thread_queue_size` 代表缓存队列的大小，它的默认值是 `-1`。在 ffmpeg 里面 `-1` 通常代表 自动判断 的意思，根据场景自动判断该用那种策略。

用户可以在 `ffmpeg.exe` 命令行里面通过 `-thread_queue_size` 选项指定 `thread_queue_size` 的大小，如果不指定，默认就是 `-1`

上图中的逻辑就是，当 `thread_queue_size` 等于 `-1`，而且输入文件个数大于 1 ，就会把 `thread_queue_size` 设置成 8，所以队列大小默认是 8

然后就会调 `av_thread_message_queue_alloc()` 来创建线程消息队列，线程消息队列 是基于 [AVFifoBuffer](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/avfifobuffer.html) 实现的，可以用阻塞或者非阻塞的方式来操作这个队列。

提示：关于多线程的封装，原理，实现，推荐阅读《[FFmpeg的多线程介绍](https://ffmpeg.xianwaizhiyin.net/api-ffmpeg/thread.html)》，本文主要讲解 `ffmpeg.c` 的多线程读文件的逻辑。

最后就会调 `pthread_create()` 函数创建 `input_thread` 线程，每个文件都创建一个线程。

----

下面就来看一些 `input_thread` 线程的内部逻辑，流程如下：

![0-3](multi_thread\0-3.jpg)

![0-4](multi_thread\0-4.png)

上图中，整个逻辑会不断读取 `AVPacket`，然后放进去 `in_thread_queue` 队列，如果塞满队列，就会阻塞等待有空间可以写入。

---

**虽然读取输入文件的 `AVPacket` 的时候是多线程的，但是处理 `AVPacket` 的时候并不是多线程的**，而是单线程，一个一个从队列里面读取出来处理的。

读取  `in_thread_queue` 队列的方法是 `get_input_packet_mt()` 函数，他的上层调用流程如下：

![0-5](multi_thread\0-5.jpg)

不过并不是如果输入文件有多个，就一定会开启多线程，如果你想不让多线程读输入文件，可以直接在命令行指定 `-thread_queue_size 0` ，如下：

```
ffmpeg.exe ^
-thread_queue_size 0 -i juren.aac ^
-thread_queue_size 0 -i juren.h264 ^
-f mp4 juren.mp4
```

只要把队列大小设置为 0 ，就不会开启多线程，也不会调 `get_input_packet_mt()` 去获取 `AVPacket`，如下：

![0-6](multi_thread\0-6.png)

最后讲解一下多线程读取输入文件的优势是什么？因为从 `mp4`，`flv` 里面读取 `AVPacket` ，跟普通的 `fread()` 读取文件内容不太一样。mp4 是有一个解复用的过程的，相对于 `fread()` ，`av_read_frame()` 函数要执行更多的指令，才能拿到  `AVPacket` 

所以如果是多线程读取，就可以并发提前去解复用。**节省的是解复用的时间。** 

不过如果输入源是 yuv 跟 pcm 数据，我个人觉得多线程的优势并不明显，因为解复用的过程很简短。因为他虽然是多线程去读，但处理的时候，还是一个一个 `AVPacket` 进行处理的。



