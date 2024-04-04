# 用msys2与msvc编译FFmpeg—FFmpeg调试环境搭建

<div id="meta-description---">用msys2与msvc编译FFmpeg</div>

本文讲解如何使用  msys2 + msvc 来编译 FFmpeg ，msys2 的安装请看 [《MSYS2介绍》](https://ffmpeg.xianwaizhiyin.net/msys2/intro.html)。

下面开始操作，先找到 **x64 Native Tools Command Prompt for VS 2019** 这个命令工具，点击它打开命令行，如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-6.png)

**这样打开的命令行 是有 vs2019 的环境变量的**，如下图：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-7.png)

千万不要用下面这种 win+R 的方式打开命令行，这样子打开命令行是没有 vs2019 的环境变量的。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-8.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-9.png)



------

为了让 msys2 能继承 vs2019 的环境变量，需要修改 `C:\msys64\msys2_shell.cmd` 中的 `rem set MSYS2_PATH_TYPE=inherit`，去掉rem，取消这⼀句的注释。使MSYS2的环境变量继承当前CMD的窗口的环境变量。

提醒：FFmpeg-4.4.1 版本已经不需要 重命名 `C:/msys64/usr/bin/link.exe` 为 `C:/msys64/usr/bin/link.bak` ， 避免和MSVC 的 link.exe 抵触。早期的版本可能需要。因为在 configure 里面编译的时候，调用的是 ./compat/window/mslink ，如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-0-1.png)

```
./compat/window/mslink 代码
#!/bin/sh

LINK_EXE_PATH=$(dirname "$(command -v cl)")/link
if [ -x "$LINK_EXE_PATH" ]; then
    "$LINK_EXE_PATH" $@
else
    link.exe $@
fi
exit $?
```

上面是 `mslink` 的代码，可以看到，他的逻辑就是优先 使用 跟 `cl.exe` 同目录下的 `link.exe`。`cl.exe` 只有在vs2019 那里才有，`C:/msys64/usr/bin` 目录下没有 `cl.exe`，所以会优选使用 vs2019 里面的 `link.exe`，所以不重命名 `C:/msys64/usr/bin/link.exe` 也没关系。

------

在 x64 Native Tools Command Prompt for VS 2019 命令窗口输入 `cd c:\msys64\` 先回到 msys64目录。

然后再输入 `.\msys2_shell.cmd -mingw64`，启动 msys2 命令行窗口，如图：

提醒：`-mingw64` 是指使用 64 位的 gcc 环境，但是本文不使用 gcc 编译器，用的是 MSVC 编译器。

由于上面使用 x64 打开的命令行窗口，所以使用的是 msvc 64 位的编译器，如果需要使用 32 位的 msvc ，要用  **x86** Native Tools Command Prompt for VS 2019 打开 msys2 环境。

```
#回到 MSYS2 的安装目录
cd c:\msys64\
#启动 msys2 命令行
.\msys2_shell.cmd -mingw64
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-10.png)

在 msys2 命令行窗口 输入 `echo $LIB` ，可以看到 msys2 命令行窗口 已经继承了 vs2019 的 lib 环境变量。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-11.png)

再输入一下 `which cl.exe` ，确认一下 cl.exe 是在vs2019的目录下，同时看下 cl.exe 的目录是否有 link.exe。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-12.png)

------

MSYS2 + MSVC 环境已经准备好了，MSYS2 命令行已经继承了 vs2019 的环境变量，下面开始编译 FFmpeg。

1，上 Github 下载 [FFmpeg-n4.4.1.zip](https://github.com/FFmpeg/FFmpeg/archive/refs/tags/n4.4.1.zip) 代码，放到 C:\msys64\home\loken\ffmpeg 目录，如下图：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-1.png)

2，回到 msys2 命令行窗口，安装所需软件，命令如下：

```
pacman -S diffutils make pkg-config yasm
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-2.png)

3，进入 FFmpeg-n4.4.1 源码目录

```
cd /home/loken/ffmpeg/FFmpeg-n4.4.1
```

4，执行 configure ，如下：

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-4.4-msvc \
--enable-gpl \
--enable-nonfree \
--enable-shared \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--toolchain=msvc

make -j8
# 要执行 make install
make install
```

5，编译完成之后，ffmepg-4.4-msvc 目录如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-3.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-4.png)

在 configure的时候 我故意使用了 --enable-shared 开启了编译动态库，所以生成了 一堆的 lib 跟 dll 文件。

上面的流程中，需要执行 make install， 因为 install 这个操作会把 exe 跟 dll 复制到同一个目录，这样才能运行。

------

这些 lib 跟 dll 都是可以被 MSVC 使用的，他们本身就是 msvc 编译出来的。具体试用，可以参考[《用msys2与mingw编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw.html)最后的 ffmpeg-test 的 version.exe 程序，使用方法是一样的。



相关阅读：

1，[《官方MSVC编译FFmpeg》](https://trac.ffmpeg.org/wiki/CompilationGuide/MSVC )
