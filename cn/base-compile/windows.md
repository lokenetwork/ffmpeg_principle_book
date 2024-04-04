# Windows编译环境介绍—编译链接基础知识

<div id="meta-description---">windows 环境下编译 C/C++ 项目，会涉及到以下4个软件，或者说是工具套件。
1，MSVC ：Windows 原生编译套件，全称 Microsoft Visual C++。vs2019 默认就是使用这个编译套件。
2，MinGW ：GCC 编译工具链 在 Windows 平台的移植。
3，Cygwin ：不仅仅是把 GCC编译工具链移植到 Windows 平台，还有相关的Linux 命令也有移植，ls，mkdir，clear 之类的命令也有移植。
4，MSYS2 ：集合 MinGW 的GCC 编译工具 链 跟 Cygwin 的配套工具。</div>

windows 环境下编译 C/C++ 项目，会涉及到以下4个软件，或者说是工具套件。

1，MSVC ：Windows 原生编译套件，全称 Microsoft Visual C++。vs2019 默认就是使用这个编译套件。

2，MinGW ：GCC 编译工具链 在 Windows 平台的移植。

3，Cygwin ：不仅仅是把 GCC编译工具链移植到 Windows 平台，还有相关的Linux 命令也有移植，ls，mkdir，clear 之类的命令也有移植。

4，MSYS2 ：集合 MinGW 的GCC 编译工具 链 跟 Cygwin 的配套工具。

------

虽然 MinGW 跟 Cygwin 都提供了 gcc 编译器，但是MinGW 的 gcc.exe 是更原生一些的，没有依赖 cygwin1.dll 。而 Cygwin 的 gcc.exe 依赖 cygwin1.dll。

Cygwin 平台大部分的软件都依赖 cygwin1.dll，这个 dll 是一个 POSIX 模拟层，模拟了很多 Linux 的函数，fork, spawn, signals, select, sockets 。

因为 Cygwin 有模拟层，所以他的兼容性跟移植性会更好一些。也就是说如果你要移植一个Linux的软件，用 Cygwin 开发效率会更高一些。

而 MinGW 的 gcc.exe 实际上跟 MSVC 的 cl.exe 是类似的，编译出来的都是原生的windows程序。

然后 MSYS2 使用的编译工具链是 MinGW 的 gcc，但是 ls，mkdir，clear 之类的命令 是基于 Cygwin 修改过来的。

MSYS2 更注重于编译生成原生的 Windows 应用，而 Cygwin 专注于移植，基于 cygwin1.dll 模拟层来移植。

例如 MSYS2 使用的 C语言运行时库 是 MSVCR，而 Cygwin 用的是 newlib 运行时。

------

提醒一下：上面 4个工具，主要都是用来 编译 生成 Windows 的 lib 静态库，dll 动态库，或者 exe 可执行文件的。他们不能生成 Linux 平台的 ELF 格式的可执行文件或者静态库动态库。也不能把 Linux 的ELF可执行文件放在这些平台运行，要运行必须用源码重新编译成 exe。

可以把上面 4 个软件看成是为了 Windows 服务的。

上面的编译器如果混用，会带来一些 ABI 兼容的问题，例如 MinGW 的 gcc 编译器生成的 dll 动态库 给 MSVC 的 cl.exe 使用。这些内容在后面都会有所讲解。

------

参考资料：

1，[《什么是msys2》](https://www.msys2.org/docs/what-is-msys2/)

2，[《使用 msys2 的软件》](https://www.msys2.org/docs/who-is-using-msys2/)

3，[《什么是 MinGW》](https://www.mingw-w64.org/)

4，[《什么是 Cygwin》](https://www.cygwin.com/)
