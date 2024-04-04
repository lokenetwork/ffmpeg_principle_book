# configure逻辑分析-C章—FFFmpeg configure源码分析

5725 ~  5753 都是一些逻辑代码，不讲解，开启动态库 pic 之类的。

------

5753 ~ 5759 调了 `nm` 命令来设置一个 `extern_prefix` 变量。`nm` 可以提取符号信息。

![configure-logic3-1-1](configure-logic3\configure-logic3-1-1.png)

运行结果如下：

![configure-logic3-1-2](configure-logic3\configure-logic3-1-2.png)

所以这个 `extern_prefix`  一般是 空，在什么场景使用我暂时也不太清楚。

------

5769 ~ 5773 是检测大小端的代码。

![configure-logic3-1-3](configure-logic3\configure-logic3-1-3.png)

------

5781 ~ 5791 是 `check_gas` 函数的实现代码。这是一个探测汇编器的函数，决定后面用哪种 汇编器来编译汇编代码。

5793 ~ 6043 都是用探测到的 汇编器 实际编译一些代码，确认汇编器可用。不同芯片指令集会用不同的汇编代码来测试。

![configure-logic3-1-4](configure-logic3\configure-logic3-1-4.png)



------

从 6048 行开始 ，就是 检测 网络组件的代码，一直到 6086 行。

因为 FFmpeg 不只是一个 处理本地文件的工具，他还可以推流拉流，所以需要网络组件。

![configure-logic3-1-5](configure-logic3\configure-logic3-1-5.png)

检测网络组件，实际上就是实际编译一下 一些 function 跟 type。

------

6096 ~ 6105 行是配置自定义的内存管理器，FFmpeg 支持 `jemalloc` 跟 `tcmalloc`。如下：

![configure-logic3-1-6](configure-logic3\configure-logic3-1-6.png)

------

从 6107 行还是，就是检测 各种 头文件，各种函数，各种库，一直到 6277行。

![configure-logic3-1-7](configure-logic3\configure-logic3-1-7.png)

不过我觉得这些代码不太好，因为没有区分环境，例如 linux 环境 肯定没有 windows.h 头文件，检测他有什么意义呢？

没有区分环境，导致 configure 执行了很多无用代码，所以 configure 有点慢。

也可能是 FFmpeg 的代码量太庞大了，不好区分环境，所以他通通检测一遍。检测通过设置为 yes，不通过 设置为 no。

------

6268 ~ 6306 是处理 线程相关的代码的，比较重要，实际上就是编译检测一些 线程 API 好不好使。

![configure-logic3-1-8](configure-logic3\configure-logic3-1-8.png)

------

6308 ~ 6593，都是各种检测，调了 enabled 函数，configure 的函数命名习惯是这样的，enable  是把变量设置为 yes ，后面加了一个 d 的函数是检测变量是否为 yes，不是yes 就返回 非 0。

------

6595 ~ 6603，是 makeinfo ，perl，pod2man 之类的代码。

这里提醒一下，因为 FFmpeg 这个项目 发展了 近20 年，所以他的一些 代码是旧代码，现在已经很少用到。包括很多 CPU 芯片指令集，已经退出主流市场了。

------

6604 ~ 6624，是检测 V4L2 的 api 函数，如下：

![configure-logic3-1-9](configure-logic3\configure-logic3-1-9.png)

------

6624 ~ 6670，也会是一些检测代码，设置为 yes 或者 no。跳过不讲。

------

6674 ~ 6689，是检测 d3d11va 的代码，可以重点阅读。

![configure-logic3-2-1](configure-logic3\configure-logic3-2-1.png)

------

6772 ~ 6784，是检测 nvcc 的代码，如下：

![configure-logic3-2-2](configure-logic3\configure-logic3-2-2.png)

------

6799 ~ 6811，是往编译器加一些比较有用的参数，如果支持的话。

![configure-logic3-2-3](configure-logic3\configure-logic3-2-3.png)

------

6849 ~ 6855，是往链接器加一些选项，如下：

![configure-logic3-2-4](configure-logic3\configure-logic3-2-4.png)

------

注意 `cflags_speed` 变量，编译器优化代码的参数都在这里。

![configure-logic3-2-5](configure-logic3\configure-logic3-2-5.png)
