# 用WinDbg断点调试FFmpeg—FFmpeg调试环境搭建

<div id="meta-description---">本文主要讲解 WinDbg 调试器的使用。WinDbg在 Windows 里面的地位，就跟 GDB 在 Linux 的地位一样。可以通过 微软的官方网站 安装 WinDbg。
WinDbg 是比较轻量级的调试工具，在一些场景下比较实用，例如不方便安装 vs2019。</div>

本文主要讲解 WinDbg 调试器的使用。[WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools) 在 Windows 里面的地位，就跟 GDB 在 Linux 的地位一样。可以通过 微软的官方网站 [下载](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools) 安装 WinDbg。

WinDbg 是比较轻量级的调试工具，在一些场景下比较实用，例如不方便安装 vs2019。

只要有 符号信息表（symbols） 跟 调试信息表（debug info），一样能用 WinDbg 进行 源码级调试 跟 各种断点调试。这些信息，windows 是放在 pdb 文件里面的，在源码目录，可以看到 ffmpeg_g.pdb 这个文件。

FFmpeg 的编译过程跟 前面文章 《[用msys2与msvc编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html) 》一样的，都是使用 msys2 + msvc。请按照之前的教程编译出来 ffmpeg.exe 文件，如果已经编译出来就可以直接用之前的 ffmpeg.exe。

![windows10-windbg-1-1](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-1.png)

WinDbg 有 32 位跟64位，我们的 ffmpeg.exe 是 64 位的，所以选用 WinDbg 64位来调试。如下：

![windows10-windbg-1-2](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-2.png)



------

打开 windbg.exe ，界面如下：

![windows10-windbg-1-3](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-3.png)

然后点击 菜单栏的 File → Open Executable，会弹出窗口，如下：

![windows10-windbg-1-4](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-4.png)

上图中 设置了 Arguments 参数 跟工作目录，参数如下：

Arguments ：`-i walking-dead.mp4 -c copy walking-dead.flv -y`

Start directory：`C:\msys64\home\loken\ffmpeg\build64\ffmepg-4.4-msvc`

walking-dead.mp4 是本书经常用到的视频素材，请下载保存好。

打开之后，界面如下：

![windows10-windbg-1-5](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-5.png)

这里简单讲解一些 WinDbg 的界面，底部是命令输入框，上图中，我输入了一个 k， 查看当前断点的调用栈。

可以看到，WinDbg 会默认停在 ntdll 模块 的 LdrpDoDebuggerBreak 函数，这是 WinDbg 的默认断点，现在还没有跑进去 ffmpeg.exe 的main函数，所以我们需要加一个断点，如下：

```
# 设置断点
bu ffmpeg_g!main
# 继续执行
g
```

注意，是 ffmpeg_g ，后面有个 _g 。 设置完断点之后，再敲入一个命令 g，g 代表 go。代码就会执行到 main 那里停下来。如下：

![windows10-windbg-1-6](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-6.png)

现在讲一下 WinDbg 常用的一些命令。

1，k ：查看函数调用栈。

2，bu ：根据符号进行断点，例如 bu ffmpeg_g.exe!main ，前面要有模块名，跟gdb 有点不一样。

3，bl：查看所有断点。

4，p：单步步进。

5，g：代码继续执行，go 的意思，快捷键 F5

WinDbg 的更多命令，请看 [《微软WinDbg 文档》](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/debugger/getting-started-with-windbg)，[《WinDbg 调试器文档》](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/debugger/bp--bu--bm--set-breakpoint-) 。

------

WinDbg 还有更多的调试窗口可以调出来，这些窗口都在菜单栏的 View 里面，这里简单介绍一下这些窗口。

1，Watch，观察窗口。点击可以添加自己想观察的全局变量或者局部变量。

2，Locals，局部变量窗口，运行到某个函数，这个窗口就是显示这个函数的局部变量信息。

3，Registers，寄存器窗口。

4，Memory，内存窗口。

4，Call Stack ，函数调用栈。

6，Disassembly，汇编代码窗口。

下面我把一些窗口调出来看看效果，如下：

![windows10-windbg-1-7](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-windbg/windows10-windbg-1-7.png)

调试器的功能都是类似的，常用的功能无非就是 数据断点，函数断点，然后可以观察变量之类的。

WinDbg 这个调试器的具体架构实现跟用法，墙裂推荐 **《软件调试》卷二 windows 下册第30章** ，通过这本书，你可以了解到如何实现一个调试器。

参考资料：

1，《软件调试》卷二 windows 下册第30章 - 张银奎
