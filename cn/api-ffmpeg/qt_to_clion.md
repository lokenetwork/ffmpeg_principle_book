# 移植Qt示例到clion调试—FFmpeg API教程

<div id="meta-description---">《FFmpeg原理》中的示例都是 Qt 工程的，有些读者本身不是做 Windows 开发，对 Qt 也不太熟悉。所以本文讲解一下，如何把这些 Qt 工程移植到 ubuntu + clion 里面进行调试。</div>

《FFmpeg原理》中的示例都是 Qt 工程的，有些读者本身不是做 Windows 开发，对 Qt 也不太熟悉。所以本文讲解一下，如何把这些 Qt 工程移植到 ubuntu + clion 里面进行调试。

阅读本文前，请先看一遍《[用Ubuntu18与clion调试FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html)》

大部分的 Qt 示例都只有一个 `main.c` 文件，所以移植起来是非常简单的。本文以 [input](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/input) 项目为例讲解，input 是一个打开输入源的示例程序。

对于初学者而言，重新写 `Makefile` 还是比较繁琐，所以我们就直接基于 FFmpeg-n4.4.1 的源码进行修改就可以。

怎么修改呢？直接把 `ffprobe.c` 的 `main` 函数替换成 `input` 项目的 `main` 函数就完事了，如下：

![1-1](qt_to_clion\1-1.png)

注意事项：`ffprobe.c` 的 `main` 函数以外的代码，例如那些变量跟函数定义，我们不要删掉它，就保留就可以了，要不会编译不通过，但是这些变量跟函数是没有被使用了。

修改完成 `main` 函数之后，我们直接 `make -j12` 就可以编译生成 `ffprobe` 可执行程序了。

然后我们修改一下 `clion` 的调试配置，如下：

![1-2](qt_to_clion\1-2.png)

然后我们在 `main` 函数里面打一个断点，就可以调试这些示例程序了，如下：

![1-3](qt_to_clion\1-3.png)

`input` 项目的运行结果如下：

![1-4](qt_to_clion\1-4.png)

总结：直接把 `ffprobe.c` 的 `main` 函数 替换成 示例程序的 `main` 函数就可以了。
