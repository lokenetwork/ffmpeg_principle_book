# 用windows10与vs2019调试FFmpeg源码—FFmpeg调试环境搭建

<div id="meta-description---">用windows10与vs2019调试FFmpeg源码</div>

在 windows 环境下，有两种**编译** FFmpeg 源码的方式。

1，MSVC，全称  Microsoft Visual C++。 MS 是微软的缩写，V 是 Visual，C代表 C++。

2，MinGW，全称  Minimalist GNU for Windows，可以看做是 Linux 编译环境在 windows 的移植。采用的编译器是 gcc。

因为 mingw 是在windows环境套一层模拟linux的函数，所以调试一些内部代码不太方便，本书不讲解 MinGW 这种编译方式，感兴趣的朋友可以看博客之前的文章《[window10_ffmpeg调试环境搭建-自己编译](https://www.xianwaizhiyin.net/?p=88)》

------

windows 环境主要有以下 2 种 **调试** FFmpeg源代码 的方法。

**1，vs2019 + VsDebug + msys2 + MSVC**

**1，WinDbg+ msys2 + MSVC**



------

本文先讲解 vs2019 + msys2 + MSVC 的调试环境搭建。

FFmpeg 的编译过程需要用到MSVC 的编译套件，cl.exe，link.exe 跟  LIBCMT.lib 等库，MSVC 是跟 vs2019 一起的，所以需要先安装 vs2019。

------

因为 windows 环境只能执行 batch 脚本，不能执行 shell 脚本。同时 windows 的 CMD 命令行，也没有 make，ls，mkdir 这些 linux 的命令。所以需要安装 [msys2](https://www.msys2.org/) ，可以把 msys2 理解成一个 linux 的环境，在 msys2 里面可以执行 shell 脚本跟很多linux的操作。

为了跟随文章的节奏，最好把 msys2 安装在 C盘， C:\msys64 的位置。

------

到这里，msys2 跟 vs2019 都安装完成了，下面的操作是最重要，非常重要的，操作错误万劫不复。

先找到  **x64 Native Tools Command Prompt for VS 2019** 这个命令工具，点击它打开命令行，如下：

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-6.png">
</div>

**这样打开的命令行 是有 vs2019 的环境变量的**，如下图：

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-7.png">
</div>



千万不要用下面这种 win+R 的方式打开命令行，这样子打开命令行是没有 vs2019 的环境变量的。

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-8.png">
</div>

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-9.png">
</div>

------

为了让 msys2 能继承 vs2019 的环境变量，需要修改  `C:\msys64\msys2_shell.cmd` 中的 `rem set MSYS2_PATH_TYPE=inherit`，去掉rem，取消这⼀句的注释。使MSYS2的环境变量继承当前CMD的窗口的环境变量。

提醒：FFmpeg-4.4.1 版本已经不需要 重命名 `C:/msys64/usr/bin/link.exe` 为 `C:/msys64/usr/bin/link.bak` ， 避免和MSVC 的 link.exe 抵触。早期的版本可能需要。因为在 configure 里面编译的时候，调用的是 ./compat/window/mslink ，如下：

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-0-1.png">
</div>

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

上面是 mslink 的代码，可以看到，他的逻辑就是优先 使用 跟 cl.exe 同目录下的 link.exe。cl.exe 只有在vs2019 那里才有，C:/msys64/usr/bin 目录下没有 cl.exe，所以会优选使用 vs2019 里面的link.exe，所以不重命名 `C:/msys64/usr/bin/link.exe` 也没关系。

------

在 x64 Native Tools Command Prompt for VS 2019 命令窗口输入 `cd c:\msys64\` 先回到 msys64目录。

然后再输入 `.\msys2_shell.cmd -mingw64`，启动 msys2 命令行窗口，如图：

提醒：`-mingw64` 是打开 64 位环境，因此编译出来的 ffmpeg 就是 64 位的，如果想编译 32位的，这里改成 `-mingw32` 即可。

```
#回到 MSYS2 的安装目录
cd c:\msys64\
#启动 msys2 命令行
.\msys2_shell.cmd -mingw64
```

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-10.png">
</div>

在 msys2 命令行窗口 输入 `echo $LIB` ，可以看到 msys2 命令行窗口 已经继承了 vs2019 的 lib 环境变量。

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-11.png">
</div>

再输入一下 `which cl.exe` ，确认一下 cl.exe 是在vs2019的目录下，同时看下 cl.exe 的目录是否有 link.exe。

<div align="center">
    <img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-12.png">
</div>

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

5，编译完成之后，ffmpeg\FFmpeg-n4.4.1 目录如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-3.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/msvc-1-4.png)

在 configure的时候 我故意使用了 --enable-shared 开启了编译动态库，所以生成了 一堆的 lib 跟 dll 文件。

上面的流程中，需要执行 make install， 因为 install 这个操作会把 exe 跟 dll 复制到同一个目录，这样才能运行。

------

现在，我们已经用 MSVC 的方式编译出来 ffmpeg 的exe文件了，这个 ffmpeg.exe 是可以直接运行的。

提醒：编译成功之后千万不要删除或者移动 FFmpeg-n4.4.1 源码目录，因为符号对应的源代码位置写进去了。移动了目录调试的时候就会找不到源代码。

Windows 下的调试工具有两个。

1，VsDebug，这个工具是 msvsmon.exe，这个工具被集成在 vs2017 ~ vs2019 里面。

2，WinDgb，这个工具就跟 Linux 的 GDB 类似。本文不讲这个工具的使用。

------

现在用 vs2019 打开文件夹 ffmpeg\build64\ffmepg-4.4-msvc ，如下：

![vs2019-1-1](windows10-vs2019\vs2019-1-1.png)

![vs2019-1-2](windows10-vs2019\vs2019-1-2.png)

![vs2019-1-3](windows10-vs2019\vs2019-1-3.png)

由于我在 configure 的时候没有指定 SDL.dll 的位置，所以 ffplay.exe 没有编译出来，这个读者自行探索。

------

上面的流程，我只是用 vs2019 打开了一个文件夹，没有创建 解决方案 （solution），这是因为 在 Visual Studio 2017 及更高版本中 ，可以不创建解决方案，直接调试 exe 文件。

不创建解决方案直接调试 exe ，依赖两个文件，如下：

1，tasks.vs.json ，这个文件 类似 clion 的 Makefile Target，就是 make ，make install，make clean。本文是在 msys2 命令行手动编译 ffmpeg 的，所以不用配置 tasks.vs.json。详情请看[《tasks.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/tasks-vs-json-schema-reference-cpp?view=msvc-170)

2，launch.vs.json，这个文件 类似 clion 的 Makefile Application，配置要调试的可执行文件，参数之类的，这个文件是本文重点。

如何创建 launch.vs.json 文件呢？首先，右边点击 ffmpeg.exe 文件，就会显示 "Debug and Launch Setting"，如下

![vs2019-1-4](windows10-vs2019\vs2019-1-4.png)

因为是 MSVC 编译的 ffmpeg ，所以选择 Native 原生的方式，如下：

![vs2019-1-5](windows10-vs2019\vs2019-1-5.png)

这样 launch.vs.json 就创建完成了，这个文件是在 .vs 隐藏目录下的，如下：

![vs2019-1-6](windows10-vs2019\vs2019-1-6.png)

再 把 ffmpeg.exe 设置为启动调试选项，如下：

![vs2019-1-7](windows10-vs2019\vs2019-1-7.png)

然后，菜单栏的绿色按钮就可以点击了。如下：

![vs2019-1-8](windows10-vs2019\vs2019-1-8.png)

现在调试 ffmpeg.exe 只会一闪而过，因为没有指定命令行参数，所以需要修改一下 launch.vs.json，内容如下：

```
{
  "version": "0.2.1",
  "defaults": {},
  "configurations": [
    {
      "type": "native",
      "name": "ffmpeg.exe help",
      "project": "bin\\ffmpeg.exe",
      "args": [ "--help" ]
    },
    {
      "type": "native",
      "name": "ffmpeg.exe mux",
      "project": "bin\\ffmpeg.exe",
      "args": [ "-i C:\\msys64\\home\\loken\\ffmpeg\\build64\\ffmepg-4.4-msvc\\walking-dead.mp4 -c copy C:\\msys64\\home\\loken\\ffmpeg\\build64\\ffmepg-4.4-msvc\\walking-dead.flv -y" ]
    }
  ]
}
```

如上，我创建了两个调试选项，一个是 打印 help信息，一个是转码 walking-dead.mp4 ，这个 mp4 文件请下载放到 ffmepg-4.4-msvc 目录下面。

更多  launch.vs.json 参数请查看微软的文档[《launch.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/launch-vs-schema-reference-cpp?view=msvc-170)。

上面的配置，因为 windows 的编解码跟 Linux 有点不一样，要指定一些具体的采样率信息，所以我直接 -c copy 不解码了。

配置之后，可以看到，菜单栏有两个调试选项可以选择，如下：

![vs2019-1-9](windows10-vs2019\vs2019-1-9.png)

咱们现在选择 ffmpeg.exe mux 来进行调试。如果直接点击 绿色三角按钮，会直接运行完毕，默认是没有断点的，所以我们需要先添加一个断点。

添加断点其实 跟 gdb 的 b main 差不多。点击菜单栏 的 Debug → Windows → Breakpoits ，即可添加断点。如下：

![vs2019-1-10](windows10-vs2019\vs2019-1-10.png)

现在再点绿色三角符号，就可以直接停在 main 函数里面，如下：

![vs2019-1-11](windows10-vs2019\vs2019-1-11.png)

现在，你可以在 ffmpeg.c 的源文件的具体行数打断点，方便调试。

由于我们没有在 vs2019 建立解决方案，是直接用 VsDebug 来调试，所以代码函数没有高亮，变量右侧也没有实时提示，只有底部一个 watch 区域可能看到变量数据。

跟之前在 ubuntu18 clion 的环境有点差距，clion 里面变量右侧是有实时信息的。如下：

![vs2019-1-12](windows10-vs2019\vs2019-1-12.png)

vs2019 其实也是可以建立解决方案，github 有个开源项目 [ShiftMediaProject](https://github.com/ShiftMediaProject/FFmpeg/tree/master/SMP)。VsDebug 跟 watch 在一般情况够用。

------

vs2019 也可以直接 调 VsDebug 直接附加到现有进程上进行调试，跟 GDB 附加进程一样。

点击 Debug → Attach to Process， 如下：

![vs2019-1-13](windows10-vs2019\vs2019-1-13.png)

参考资料：

1，[《自定义“打开文件夹”开发的生成和调试任务》](https://docs.microsoft.com/zh-cn/visualstudio/ide/customize-build-and-debug-tasks-in-visual-studio?view=vs-2022)

2，[《launch.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/launch-vs-schema-reference-cpp?view=msvc-170)

