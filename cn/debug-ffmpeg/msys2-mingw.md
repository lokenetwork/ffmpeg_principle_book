# 用msys2与mingw编译FFmpeg—FFmpeg调试环境搭建

<div id="meta-description---">用msys2与mingw编译FFmpeg</div>

FFmpeg 官网提供了两种编译源码的方法。

1，msys2 + mingw （本文的编译方法）

2，msys2 + msvc

MinGW 在前面文章[《MinGW介绍》](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-intro.html)已经介绍过了，MinGW 就是 把 gcc 编译器移植到 windows 了，并且提供了一些 以前只有在 Linux 才有的函数，例如 pthread_create() 。

------

为什么需要安装 msys2 ？

因为 windows 环境只能执行 batch 脚本，不能执行 shell 脚本。同时 windows 的 CMD 命令行，也没有 make，ls，mkdir 这些 linux 的命令。所以需要安装 msys2，可以把 msys2 理解成一个 linux 的环境，在 msys2 里面可以执行 shell 脚本跟很多linux的操作。

------

msys2 的安装请看 [《MSYS2介绍》](https://ffmpeg.xianwaizhiyin.net/msys2/intro.html)，下面开始操作，**打开普通的命令行窗口**，执行以下命令，进入 msys2 环境：

```
cd C:\msys64
.\msys2_shell.cmd -mingw64
```

上面的 `-mingw64` 是使用 64位的 gcc，如果需要使用 32 位 gcc，可以用 `-mingw32`

在 msys2 环境下，安装一些 必要的软件：

```
# 刷新软件包数据
pacman -Sy  
# 安装mingw-w64。
pacman -S mingw-w64-x86_64-toolchain
pacman -S git
pacman -S make
pacman -S automake 
pacman -S autoconf
pacman -S perl
pacman -S libtool
pacman -S mingw-w64-x86_64-cmake
pacman -S pkg-config 
pacman -S yasm
pacman -S diffutils
# 编译x264 需要 nasm
pacman -S nasm
```

小技巧， pacman -Ss 关键字：在仓库中搜索含关键字的包。

------

然后上 Github 下载 [FFmpeg-n4.4.1.zip](https://github.com/FFmpeg/FFmpeg/archive/refs/tags/n4.4.1.zip) 代码，放到 下图中的目录，这样 msys2 环境也能找到。

![msys2-mingw-1-1](msys2-mingw\msys2-mingw-1-1.png)

![msys2-mingw-1-2](msys2-mingw\msys2-mingw-1-2.png)

------

进入 FFmpeg-n4.4.1 目录，开始编译，命令如下：

```
cd /home/loken/ffmpeg/ffmpeg-n4.4.1

./configure \
--prefix=/home/loken/ffmpeg/build64/ffmpeg-n4.4.1-mingw \
--enable-gpl \
--enable-debug=3 \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--enable-nonfree \
--enable-shared

make -j8
make install
```

上面的命令执行完之后，`ffmpeg-n4.4.1-mingw` 目录的结构如下：

![msys2-mingw-1-1](msys2-mingw\msys2-mingw-1-3.png)

可以看到，FFmpeg 的 makefile 特别智能，帮我们生成了 lib 导入库，这样就能很方便被 msvc 使用。

我们用 `DependenciesGui.exe` 查看一下 `ffmpeg.exe` 的整体依赖情况

![msys2-mingw-1-3-1](msys2-mingw\msys2-mingw-1-3-1.png)

上图中，圈出来的 dll 都是 mingw 里面的 dll。因为 mingw 编译出来的 ffmpeg 依赖这些 mingw 的库，所以只能在 msys2 环境下跑 `ffmpeg.exe` ，如果想在 WinCMD 创建运行 `ffmpeg.exe` ，需要把这些依赖的库拷贝到 跟 `ffmpeg.exe` 同级的目录。

------

我们可以 使用 以下命令 查看 `avutil-56.dl`l 导出了哪些函数给我们使用。

```
dumpbin /EXPORTS avutil-56.dll > avutil-56.txt
```

![msys2-mingw-1-4](msys2-mingw\msys2-mingw-1-4.png)

从里面找到了一个 av_version_info 函数，注意那个地址 0005C570 。

------

我们现在测试一下 这个 avutil 动态库 在 MSVC 编译环境下好不好用。在 D 盘新建一个 目录 ffmpeg-test ，再创建一个 `version.c` 文件，内容如下：

```
#include <stdio.h>
const char *av_version_info(void);
int main()
{
    printf("Hello FFMPEG, version is %s\n", av_version_info());
    return 0;
}
```

上面 `av_version_info` 函数 的声明是我从 `avutil.h` 头文件里面扣出来的，这样不用引入整个 `avutil.h` 头文件，相对简单一些。

把 `avutil.lib` ，`avutil-56.dll`  也复制 到 ffmpeg-test  目录。

把 `C:\msys64\mingw64\bin` 目录下的 `libwinpthread-1.dll` 也拷贝到 ffmpeg-test 目录。如下：

![msys2-mingw-1-5](msys2-mingw\msys2-mingw-1-5.png)

上面这样做是因为 `avutil-56.dll` 依赖 `libwinpthread-1.dll` 。

------

然后 打开 vs2019 x64 的命令窗口，进入 ffmpeg-test 目录。

![msys2-mingw-1-6](msys2-mingw\msys2-mingw-1-6.png)

执行以下命令开始编译：

```
cl.exe /c version.c
link.exe /DEBUG /OUT:version.exe version.obj avutil.lib
```

![msys2-mingw-1-7](msys2-mingw\msys2-mingw-1-7.png)

因此，MinGW 的 gcc 编译出来的 FFmpeg 动态库是好使的。

------

之前 dumpbin 的时候，`av_version_info` 函数的左边有个 0005C570 地址。这个是什么东西呢？

现在我们用 WinDbg 断点调试 一个 `version.exe` 文件 ，用 `bu version!main` 打一个断点，如下：

![msys2-mingw-1-8](msys2-mingw\msys2-mingw-1-8.png)

没看出 0005C570 跟 调试器的哪个地址有关系，暂时不管。
