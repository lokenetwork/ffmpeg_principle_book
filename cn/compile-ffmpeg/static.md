# FFmpeg静态编译—FFmpeg编译教程

<div id="meta-description---">如何编译出 不依赖任何一个动态库的 ffmpeg.exe </div>

在以往的文章中，编译出来的 `ffmpeg.exe` 都不是完全静态的，总会依赖一些动态库，例如 `libm.dll` ，`libz.dll`，`libc.dll` 等等。

虽然 FFmpeg 的 `configure` 脚本 有一个 `--enable-shared` 选项，但这个选项只是 决定要不要生成 FFmpeg 的 [8个API库](https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/intro.html)。不用  `--enable-shared` 选项 就是不生成 8 个 `dll` 动态库。

但是即使不用 `--enable-shared` 选项 ，编译出来的 `ffmpeg.exe` 还是会依赖 `libm.dll` ，`libz.dll`，`libc.dll` 等等动态库。

有时候我们为了兼容性，往往希望生成一个 不依赖任何一个动态库的 `ffmpeg.exe` ，这个怎么做呢？

------

在介绍 **FFmpeg静态编译** 的方法之前，先简单介绍一下，编译，链接 的基本概念。

**1，**所有代码在编译阶段都是独立编译，直接没有关联，例如 `xx.c` 跟 `yy.c` 没有关联，即使 `xx.c` 里面用了 `yy.c` 的函数，在编译阶段也是两个文件也没有关联的。都是独立编译。

**2，**静态库只是目标文件的集合，只是简单打包一下，可以简单理解为合成一个 zip 压缩包。目标文件是指 `.o` 或者 `.obj` 后缀的文件

**3，**链接阶段，就是把所有的 `.obj` 文件链接在一起，同时还会链接一些 `lib` 后缀文件，`lib` 后缀文件可以是动态库的导入库，或者是静态库

------

这里面有一个重点就是，动态库 跟 静态库，是可以混合使用的，在 [《Linux环境混合使用静态库与动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-mix.html) 一文已经演示过的。

例如 A 库 依赖 C 库，B 库依赖 C 库。而我们的 exe 项目同时需要 A库 B库 跟 C 库。这种场景有几种混合链接的方式。

**1，**ABC 库都是 **动态链接**。

**2，**ABC 库都是 **静态链接**。

**3，**A 库是静态链接，而 B 跟 C 库是动态库链接。

**4，**A B 是静态链接， C 库是动态链接。

上面这 4 种链接方式 都是可以的，用 `-Wl,-Bstatic` 跟`-Wl,-Bdynamic` 选项即可。

但是，但是现实场景很少这么用。我们很少手动敲 命令选项传给 链接器，通常一个项目都是用 `pkg-config` 这种工具来管理链接选项的。

`pkg-config` 工具依赖一个 `pc` 后缀文件，这个 pc 文件只有两种选项，**静态选项** 跟 **动态选项**。

也就是 如果一个 B 库依赖 C 库，那 B 库的 pc 文件就只有两种选项。

**1，**B库 C库 都是静态链接。

**2，**B库 C库 都是动态链接。

很少会有一个 pc 文件是混合使用 动态 跟 静态的。虽然 链接器支持 混合使用。你也可以改 pc 文件实现混合使用，但是相对麻烦。因为  `-Wl,-Bstatic` 跟`-Wl,-Bdynamic` 选项 是有顺序的，可能会影响到其他的 pc 文件的选项。

------

FFmpeg 项目 也是使用 `pkg-config` 工具来提取链接选项的，所以 FFmpeg 的静态编译命令如下：

`gcc` 跟 `msvc` 指定静态链接的选项是不太一样的，`gcc` 是  `-static` ，而 `msvc` 是 `-MT` 。 

```
# gcc 
./configure \
--extra-cflags="-static" \
--extra-ldflags="-static" \
--pkg-config-flags="--static" \

# msvc 
./configure \
--extra-cflags="-MT" \
--pkg-config-flags="--static" \
--toolchain=msvc
```

在执行 这条命令之前，需要安装一些必备的静态库，因为操作系统为了节省硬盘空间，默认可能只有动态库，一些静态库需要手动安装的。

```
# ubuntu 系统命令
apt-get install libc6-dev
sudo apt-get install build-essential

# centos 系统命令
yum install glibc-static
yum install libstdc++-static
```

------

以上方法，在 ubuntu 跟 centos 系统里面是可以编译出不依赖任何动态库的 `ffmpeg` 命令的，如下：

![0-0-1](static\0-0-1.png)

![0-0-2](static\0-0-2.png)

------

但是在 MinGW 的环境下，只是不会依赖 MinGW 相关的动态库，但还是会依赖 Windows 系统本身的动态库，如下：

![1-1](static\1-1.png)

![0-1](static\0-1.png)

不过由于不依赖 MinGW 环境的动态库，所以我们能够在 普通 的 CMD 环境下使用 `ffmpeg.exe` 了，以前要在 msys2 命令行里才能用  `ffmpeg.exe` 

------

在 `msys2 + msvc` 环境里面，由于使用了 `-MT` 选项，所以不依赖 `msvcrt.dll` 动态库，这是 C语言的运行时库，但是还是会依赖一小部分的 `dll`，不过这些 `dll` 早就是 Windows 自带的。所以即使依赖也不会有太多的兼容问题。

![1-2](static\1-2.png)

