# MSYS2介绍

<div id="meta-description---">MSYS2介绍</div>

请先安装 MSYS2 ，[点击下载](https://www.msys2.org/)。官网有教程，按照官网的教程安装在 `C:\msys64` 里。

如何启动 msys2 ？ 打开WinCMD 进入  `C:\msys64` 目录，执行以下命令 进入 mingw32 或者 mingw64位环境。

1，`.\msys2_shell.cmd -mingw32`

2，`.\msys2_shell.cmd -mingw64`

![intro-1-0](intro\intro-1-0.png)



------

msys2 实际上是一个 在 Windows 系统模拟 Linux 的一个命令窗口程序，如下：

![intro-1-1](intro\intro-1-1.png)

**扩展知识：msys2 的这个命令行窗口是使用 [mintty](https://mintty.github.io/) 实现的。**

大部分的Linux 的命令在 msys2 的环境都有，但是msys2的这些命令其实都是一个 exe 文件，你可以在 原生的 WinCMD 窗口执行他们，如下：

![intro-1-2](intro\intro-1-2.png)

![intro-1-3](intro\intro-1-3.png)

------

msys2 里面使用默认的编译工具链是 [MinGW](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-intro.html) 的 gcc，看一下 msys64 里面的文件，发现他里面有个 mingw64 的文件夹，如下：

![msys2-1-1](msys2\msys2-1-1.png)

这些命令，就是从 MinGW 项目拷贝过来的。

------

msys2 还参考 Arch Linux 开发了 pacman 包管理器。类似 ubuntu 的 apt 命令。只需一个命令即可安装软件。









