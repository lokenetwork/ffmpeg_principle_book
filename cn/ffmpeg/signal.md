# FFmpeg的信号处理—ffmpeg.c源码分析

<div id="meta-description---">FFmpeg的信号处理</div>

`ffmpeg` 配置 信号处理函数是在 `term_init()` 函数里面的，他的函数调用位置如下：

![1-1](signal\1-1.jpg)

这个函数比较简单，它大部分的信号，都是用同一个函数 `sigterm_handler()`来处理。如下：

![1-2](signal\1-2.png)

我们平时用的最多的信号，就是这个 `SIGINT`，因为在 `ffmpeg` 命令运行过程中，当你想终止任务的之后，就会去按 `ctrl+c`，这时候就会向 `ffmpeg` 的进程发送一个 `SIGINT` 信号。

不过 `ffmpeg` 的开发者考虑得非常周到，为了防止你按错 `ctrl+c` 终止任务，他对次数进行了判断，只有接受到 3 次以上 `SIGINT` 信号才会终止任务，如下：

![1-3](signal\1-3.png)

因此，如果你想终止一个正在运行的 `ffmpeg` 任务，需要按 4 次 ctrl+c ，只按 1~3 次是没用的。
