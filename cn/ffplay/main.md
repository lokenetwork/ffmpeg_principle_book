# main入口函数分析—ffplay.c源码分析

<div id="meta-description---">FFplay main入口函数分析</div>

在开始讲解之前，分享一些阅读 项目代码的经验。无论学习哪方面的知识，都是需要正反馈才能继续学下去。在学习开源项目的时候，如果不掌握一些比较好的方法，会比较难拿到正反馈，或者要坚持学习很久才能拿到正反馈。

我个人学习开源项目的方法如下：

**1，配置好项目的调试环境。**

调试环境 优先选择 `clion`，`vs2019`，`qt creator`，次要选择 `gdb`，`WinDbg`。

调试器是你攻克开源项目的神兵利器，调试器 会把 所有 数据结构的变换，逻辑流转，完完整整地呈现在你的面前，特别是 一些 集成开发环境，变量，内存，寄存器的观察会更加容易方便。

**2，找到项目的一个最简单的场景去切入分析。**

一个庞大的开源项目，通常会有非常多的逻辑分支，流程流转，也有非常多的场景应用。撒胡椒面 式地阅读代码，会容易迷失在错综复杂的代码逻辑里面，拿到正反馈的时间越来越长。

对于一个庞大的项目，一个比较有效的方法是找到一个简单的场景，一条直线地追踪下去，那些旁路侧枝，不会跑进去的条件，就不要浪费时间去看。

等理解完一个简单的场景之后，再回去看那些有点关联的 旁路侧枝，就会更容易看懂。

本章节 采用 `clion` 来调试 `FFplay` 的代码，推荐阅读[《用Ubuntu18与clion调试FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html)

------

本章分析 `FFplay` 播放器，也是采用一个**最简单的场景**切入去分析，也就是简简单单播放一个视频文件，命令如下：

`juren-5s.mp4` 的下载地地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/ffplay/juren-5s.mp4)

```
ffplay -i juren-5s.mp4
```

上面的命令中，我没有用其他的命令参数，例如设置窗口大小，硬件加速，滤镜，等等这些参数。因为在此时此刻，这些多余参数，就是**旁路侧枝**。

我们只需先搞懂，`ffplay` 是怎么 从 `mp4` 文件读取数据，又是如何解码，如何播放的。下面正式开始：

------

`ffplay.c` 里面`main()` 入口函数的流程图如下：

![main-1-1](main\main-1-1.jpg)

代码如下：

![main-1-1-1](main\main-1-1-1.png)

![main-1-1-2](main\main-1-1-2.png)

------

上面的流程 分两部分讲，非重点函数 跟 重点函数。

**非重点函数如下：**

**1，**`init_dynload()`，设置动态库加载规则，这是一个安全函数，在 Windows 系统，默认会从当前目录加载 DLL，这容易被攻击。这个函数就是把当前目录的路径从加载规则里面去掉，里面调的是 `SetDllDirectory("")`。

**2，**`show_banner()`，打印 `ffplay` 这个软件的 版权，版本之类的信息。可以删掉他，让控制台更简洁。

------

**重点函数如下：**

**1，**`parse_options()`，解析命令行参数，虽然这是一个重点函数，但是为了力求简单，我会一笔带过。本文只用到 一个 `-i` 参数，所以这个函数在这里的作用就是设置 `input_filename` 全局变量。在 [《parse_options函数分析》](https://ffmpeg.xianwaizhiyin.net/ffplay/parse_options.html)会详细讲解 命令行解析。

**2，**`SDL_CreateWindow()`，创建 SDL 窗口，具体请看 [SDL官方文档](https://wiki.libsdl.org/Tutorials)。

**3，**`stream_open()`，这个函数是**重中之重**，上图中可以看到，可能会有 4 个线程从 `stream_open()` 里面诞生。先来讲一下这 4 个线程的作用。

`read_thread()` ：从 网络或者硬盘里面读取 `AVPacket`，读取到之后放进去 `PacketQueue` 队列。

`audio_thread()` ：从  `PacketQueue audioq` 队列拿 `AVPacket`，然后丢给解码器解码，解码出来 `AVFrame` 之后，再把 `AVFrame` 丢到 `FrameQueue` 队列。

`video_thread()` ：从  `PacketQueue videoq` 队列拿 `AVPacket`，然后丢给解码器解码，解码出来 `AVFrame` 之后，再把 `AVFrame` 丢到 `FrameQueue` 队列。

`subtitle_thread()` ：字幕线程，由于 `ffplay` 的字幕播放有点不完善，不必关注。

上面的 4 个线程 不一定会创建，如果 mp4 文件里面没有音频流，就不会创建 `audio_thread()` 线程，其他的线程 类推。

这 4 个线程之间的关系如下：

![main-1-2](main\main-1-2.jpg)

**`read_thread` 是生产者，而 `audio_thread` 跟 `video_thread` 是消费者。**

------

最后一个重点函数是 `event_loop()`，这个函数是一个死循环，主要的任务就是不断 **处理键盘按键事件** 跟 **播放视频帧**。推荐阅读 [《event_loop函数分析》](https://ffmpeg.xianwaizhiyin.net/ffplay/event_loop.html)。

至此，`ffplay.c` 的 `main()` 函数就讲解完毕了，由于 `FFplay` 播放器的函数封装都做得不错，所以 `main` 函数看起来是非常简洁的。主要功能都在各个子函数里面完成。



