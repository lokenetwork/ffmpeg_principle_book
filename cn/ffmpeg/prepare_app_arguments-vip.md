# prepare_app_arguments源码分析—ffmpeg.c源码分析

<div id="meta-description---">prepare_app_arguments源码分析</div>

ffmpeg 命令行是一个跨平台的可执行文件，可以在 Linux，Windows 上运行不出问题，这是因为在 `ffmpeg.c` 里面做了很多兼容处理，而 `prepare_app_arguments()` 函数就是其中之一。

`prepare_app_arguments()` 函数在 Linux 上是一个空函数，但是在 Windows 上却有 40 行代码，这些代码主要是用来把 **宽字符** 转成 **UTF-8** 的。

Windows 上的**宽字符**好像是 UTF-16 ，具体宽字符的规则跟标准，我没怎么去看，我也不太懂。不过无论是那种字符编码，都是一段内存数据。

下面让我们一起学习一下 `prepare_app_arguments()` 函数。

要断点调试 `prepare_app_arguments()` 函数，推荐使用 `VsDebug`，请看《[用VsDebug断点调试FFmpe](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/vsdebug.html)g》。

![1-1](prepare_app_arguments\1-1.png)

`prepare_app_arguments()` 函数一开始就用 [GetCommandLineW](https://learn.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-getcommandlinew) 函数获取整个命令行字符串的数据，然后用 [CommandLineToArgvW](https://learn.microsoft.com/zh-CN/windows/win32/api/shellapi/nf-shellapi-commandlinetoargvw) 分析这个字符串，弄成指针数组，例如根据空格分割成数组。

我也不太清楚原始的数据 `argv_ptr` 是什么字符集，不过我们可以查看一下 `argv_ptr` 与 `argv_w` 的内存数据，看看是否有差异。

![1-2](prepare_app_arguments\1-2.png)

上图中的内存数据，0x43 就是 C 字符的 ASCII码，但是转出来的宽字符，似乎在每个 ASCII码 后面都加了 00。好像是为了补够 16 位，所以宽字符应该就是 UTF-16 字符串。

可以看到，内存数据不一样，确实是两种不同的字符集，所以原始的 `argv_ptr` 肯定不是宽字符。

我猜测，`argv_ptr` 的字符集，应该跟 CMD 窗口设置的默认字符集一样，我的 应该是 UTF-8，这个后面我们验证这个猜想。

---

然后，后面的代码会申请大小适合的内存来存放 UTF-8 字符数据，如下：

![1-3](prepare_app_arguments\1-3.png)

`WideCharToMultiByte()` 函数的用法看微软官方文档就行。

上图的代码细节我们也不用太深究，就看一下他转出来的 UTF-8 的字符串内存，跟原始的 `argv_ptr` 的内存比对一下，看看是不是不一样。

![1-4](prepare_app_arguments\1-4.png)

可以看到，字符那块内存数据是一样的，所以在我的场景下，prepare_app_arguments() 函数什么都没干，只是把 原始数据 转成 宽字符数据，再转成 UTF-8，但原始数据在我的环境下，本身就是 UTF-8。

补充：也不能说是把 原始数据 转成 宽字符数据，他是直接用了 `GetCommandLineW` 函数，以宽字符的形式来获取命令行字符数据。

所以，`prepare_app_arguments()` 函数，是为了处理在 CMD 环境下，默认字符集设置成 非 UTF-8 的情况。

使用了 `prepare_app_arguments()` 函数之后，无论CMD 的字符集是什么，都默认用 `GetCommandLineW` 函数，以宽字符的形式来获取命令行字符数据，然后转成 UTF-8 字符集来处理。

