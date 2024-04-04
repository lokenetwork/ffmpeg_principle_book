# FFmpeg 的多线程API—FFmpeg API教程

<div id="meta-description---">FFmpeg 的跨平台多线程原理</div>

在 `ffmpeg.c` 里面有使用 `pthread_create`，`pthread_join` 函数，熟悉 Linux API 的朋友应该知道，这是 Linux 平台的线程函数。

但是，有趣的是，在 `Windows` 的 `msvc` 编译环境下，也能顺利编译 `ffmpeg.c` 代码文件。Windows 系统明明没有提供  `pthread_create`，`pthread_join` 函数。

`Windows` 平台的线程API 是 `CreateThread`，`_beginthreadex` 等函数。

那为什么 `ffmpeg.c` 在 `Windows` 也能使用 `pthread_create`，这就是本文的内容，**FFmpeg 的跨平台多线程原理**。

---

FFmpeg 项目对 3种 操作系统平台的线程 API 进行了封装，然后用宏来控制函数在不同平台上的实现。

1，`w32pthreads.h`，这个头文件对 Windows 系统的线程API 进行了封装。

2，`os2threads.h`，这个头文件对 OS2 类系统的线程API 进行了封装。

3，`pthread.h`，这个头文件本身就由 Linux 操作系统提供，不用封装。

主要就是 `w32pthreads.h`，`os2threads.h`，这两个头文件，对原生线程API进行封装，提供了 `pthread_create`，`pthread_join`，以及一些互斥锁的函数。

---

本文以  `w32pthreads.h` 的代码来讲解封装原理，重点代码如下：

![1-1](thread\1-1.png)

上图中，Windows 下的 `pthread_create()` 函数就是基于 `CreateThread`，`_beginthreadex` 函数实现的。

`CreateThread` 是 Win32 的函数，`_beginthreadex` 是Windows 平台的 C运行时库的函数，微软文档建议，优先使用 `_beginthreadex`，而不是 `CreateThread`

而 `pthread_join()` 函数是基于 `WaitForSingleObject` 实现的，这是 Windows 平台的 API 函数。

可能读者朋友们会疑惑，定义这么 `pthread_create` 一个函数名称，会不会冲突？

答：不会，因为他用了 `static` 限制这个函数在**本文件**使用，注意他的函数实现是直接写在头文件里面的，然后通过 `include` 包含到其他文件。

我这里的**本文件**是指，`include` 合并之后的文件，`include` 是在编译之前的预处理操作，把多个文件合并成一个文件再编译。

---

下面我们通过一个示例来演示一下，在 `msvc` 环境下如何使用 `FFmpeg` 封装的 `pthread` 函数，本文的代码下载地址：[GitHub-Thread](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/thread)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

![1-2](thread\1-2.png)

示例里面演示了如何创建线程，如何 `join` 线程，如何使用互斥锁。

线程间通信可以通过 `AVThreadMessageQueue` 来完成，`AVThreadMessageQueue` 是基于 `AVFifoBuffer` + `pthread_cond_wait` 实现的。

`AVThreadMessageQueue` 是有 [Cigaes](https://github.com/FFmpeg/FFmpeg/commits/master/libavutil/threadmessage.c) 在 2014 年提交的，如下：

![1-3](thread\1-3.png)

---

补充，默认安装 `FFmpeg` 的时候，是没有安装 `libavutil/thread.h` 头文件的，所以你需要手动复制过去。

`libavutil/thread.h` 依赖根目录的 `config.h`，`compat/w32pthreads.h`，以及 `libavutil/internal.h`，`libavutil/libm.h`，`libavutil/timer.h`

上面这些文件你都要自己拷贝过去 才能在 windows 平台正常使用 `pthread_create` 等函数。

不过也可以直接运行本文的示例项目，因为这些工作都已经做好了。

