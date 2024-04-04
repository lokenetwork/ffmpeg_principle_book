# FFmpeg引入SDL扩展—FFmpeg编译教程

<div id="meta-description---">Windows10 系统下 FFmpeg与SDL的编译教程，包括 MSVC 跟 MinGW 两种方式</div>

本文是在 msys2 环境下进行操作的，不熟悉 msys2 的可以看 [《MSYS2介绍》](https://ffmpeg.xianwaizhiyin.net/msys2/intro.html)。

FFmpeg 引入 SDL 扩展实际上非常简单，原理就是 编译 FFmpeg 的时候加上 SDL 的导入库就行。跟其他的 C/C++ 项目引入外部动态库是一样的。

SDL动态库的安装也非常简单，因为 msys2 的云仓库本身就有 SDL 的安装包，所以我们不需要下载 SDL 的源码自己编译，直接使用以下命令安装 SDL 即可。

```
pacman -S mingw-w64-x86_64-SDL2
```

 FFmpeg-n4.4.1 需要的 SDL 版本是 小于 2.1.0 的，所以我们用下面的命令确认一下 刚刚安装的 SDL 版本。

```
pacman -Qs mingw-w64-x86_64-SDL2
```

![0-1](sdl\0-1.png)

安装完成之后，可以使用以下命令查看安装了什么东西到我们的电脑。

```
pacman -Ql mingw-w64-x86_64-SDL2
```

![sdl-1-1](sdl\sdl-1-1.png)

上图中，安装了很多头文件，静态库，动态库。但只要注意以下几个文件即可。

1，`/mingw64/lib/libSDL2.a`，SDL 静态库。

2，`/mingw64/lib/libSDL2.dll.a`，SDL 动态库的**导入库**。windows 系统动态库分为两部分，导入库跟动态库，而 Linux 全部信息集成在 一个 so 文件里面。

3，`/mingw64/bin/SDL2.dll`，SDL 动态库，运行的时候需要用到，编译链接时不需要这个文件。

------

上面这个libSDL2.dll.a 跟 SDL2.dll **应该**可以被 MinGW 的编译跟 MSVC 的编译器使用，都是ABI兼容的，没有问题。但是 静态库 libSDL2.a 只能被 MinGW 使用，因为他这个静态库，只是一堆 obj 文件的集合，而这些 obj 是 MinGW的 gcc 编译出来的，gcc 编译的目标文件格式，不能被 MSVC 的 link.exe 链接器识别出来。至少目前是这样的。

如果需要在 MSVC 环境使用 SDL 静态库，只能自己用 MSVC 编译器重新编译一次 SDL 源码生成静态库。

**重点：ABI 兼容的动态库可以被不同的链接器使用，但是静态库不行。**



------

下面先讲解一下 MinGW 的环境下编译 FFmpeg 的时候如何引入 SDL 动态库。

其实步骤是跟[《用msys2与mingw编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw.html)一样的，只要在 configure 的时候加上 `--enable-sdl2` 即可，下面就来分析一下 加了这个 `--enable-sdl2` 会影响哪些编译过程。

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmpeg-n4.4.1-mingw \
--enable-gpl \
--enable-debug=3 \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--enable-nonfree \
--enable-sdl2 \
--enable-shared
```

ffbuild 目录下的 config.log 记录了整个检测的编译过程，如下：

![sdl-1-2](sdl\sdl-1-2.png)

从上图可以看出，执行了一句命令，如下：

```
pkg-config --exists --print-errors "sdl2 >= 2.0.1 sdl2 < 2.1.0"
```

这条命令是检测 SDL 的版本，要大于 2.0.1 小于 2.1.0，可以把上面的命令在 命令行执行一遍，是不会报错的，因为 我们刚开始安装的 SDl 版本是符合的。

但是大家可以把 版本改大，改成必须大于 2.3.1，立即就会报错，如下：

![sdl-1-3](sdl\sdl-1-3.png)

那这个 pkg-config 命令是从哪里检测这个版本的呢？

实际上是查看那些 .pc 后缀的文件，pkg-config 命令会在哪些路径搜索 pc 文件呢？其实就是 `$PKG_CONFIG_PATH` 变量。这个环境变量的内容如下：

![sdl-1-4](sdl\sdl-1-4.png)

可以看到都是 mingw64 目录，这是因为我 开启 msys2 命令行使用了 -mingw64 ，所以环境变量 `$PKG_CONFIG_PATH` 会设置成 64位的路径。

可以看到，在 `/mingw64/lib/pkgconfig` 目录下有个 **sdl2.pc** 文件，这个文件也是之前执行 pacman 安装 SDL 的时候一起安装的。

**sdl2.pc** 的内容如下：

```
# sdl pkg-config source file

prefix=/mingw64
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: sdl2
Description: Simple DirectMedia Layer is a cross-platform multimedia library designed to providew level access to audio, keyboard, mouse, joystick, 3D hardware via OpenGL, and 2D video framebur.
Version: 2.0.20
Requires:
Conflicts:
Libs: -L${libdir}  -lmingw32 -lSDL2main -lSDL2 -mwindows
Libs.private: -lmingw32 -ldinput8 -lshell32 -lsetupapi -ladvapi32 -luuid -lversion -loleaut32 -l32 -limm32 -lwinmm -lgdi32 -luser32 -lm  -Wl,--no-undefined -lmingw32 -lSDL2main -lSDL2 -mwindow
Cflags: -I${includedir}/SDL2  -Dmain=SDL_main
```

可以看到，上面有个 `Version: 2.0.20`， pkg-config 命令就是读取这段内容来匹配版本号的，**sdl2.pc 里面还有一些 Libs 跟 Cflags 参数，这些参数其实后面都会传递给编译器跟链接器。**

------

pkg-config 命令匹配版本号的原理已经讲解完毕，下面分析引入了 SDL 之后， gcc 的参数变化，如下：

![sdl-1-5](sdl\sdl-1-5.png)

从上图可以看到， gcc 主要多了几个参数。

1，`-IC:/msys64/mingw64/include/SDL2`，这个是指定头文件的搜索路径，这个路径就是 从 sdl2.pc 里面拿到的，注意看 sdl2.pc 里面的 Cflags 参数。

2，`-LC:/msys64/mingw64/lib`，这个是指定 动态库的导入库，或者静态库的搜索路径，注意看 sdl2.pc 里面的 Libs 参数。

提示：上面的 gcc 其实用 -c 指定只编译不链接，实际上不需要加 -L 指定库的搜索路径 ，cl.exe 根本没有 -L 选项，只有 link.exe才有 -L 选项，FFmpeg 的configure 脚本为了方便全加上去的而已。这样不会报错，只是会报一下 warning。

------

到这里，FFmpeg 引入 SDL 的原理已经很明显，就是在 gcc 的时候加一些参数，跟其他的 C/C++ 项目引入外部库是一样的。

configure 脚本跑完之后，再执行 make 跟 make installl ，就会发现 ffplay.exe 编译出来了，因为 ffplay.exe 依赖 SDL，如下：

![sdl-1-6](sdl\sdl-1-6.png)

------

从上图可以看到，ffplay.exe 是以动态库的方式引入 SDL2.dll 的，这是因为 gcc 编译的时候 如果 `-lSDL2` 这样指定外部库，gcc 会优先选择动态库的引入方式。

我们 pacman 安装 sdl 的时候，是有一个 libSDL2.a 静态库，如果我们想把静态库编译进去 ffplay.exe 应该如何做呢？

其实静态库的引入方式我觉得是比动态库要麻烦，或者说更复杂一点。

之前在 [《编译链接基础知识》](https://ffmpeg.xianwaizhiyin.net/base-compile/intro.html)一章说过，无论是 Linux 的静态库，还是 Windows 的静态库，都只是 目标文件的集合，也就是简单地把 目标文件打包在一起，并不会对里面的函数引用做重定位（地址修正）。

提示：目标文件是指 .o 或者 .obj 后缀的文件。

由于静态库不会对目标文件里面的函数引用做重定位，所以一个静态库通常只有他自己本身的代码。

举个例子，a.lib 静态库依赖 b.lib 静态库的函数，当你引入 a.lib 的时候要同时引入 b.lib。 

a.dll 动态库依赖 b.dll ，当你引入 a.dll 动态库的时候，只需要给链接器引入一个 a.dll 动态库的导入库即可，因为 a.dll 生成的时候他已经跟 b.dll 绑定了关系，所以你不需要再引入 b.dll 的导入库。

------

下面我们来看一下 SDL2.dll 这个动态库依赖哪些动态库，如下：

![sdl-1-6-2](sdl\sdl-1-6-2.png)

从上图可以看到，SDL2.dll 依赖 gdi32.dll，Setupapi.dll 等等 12个动态库，但是我们编译 FFmpeg 引入 SDL2.dll 的时候，只需要传一个导入库  libSDL2.dll.a 给 链接器即可。SDL2.dll 依赖的 Setupapi.dll 等库，我们可以不用管。

------

现在要引入 `libSDL2.a` 静态库，那我们就需要管 `libSDL2.a` 所依赖的外部库，因为静态库是没做地址修正的，没跟依赖的外部库绑定关系。

sdl.pc 文件其实记录 SDL2 依赖的外部库有哪些，如下：

```
Libs.private: -lmingw32 -ldinput8 -lshell32 -lsetupapi -ladvapi32 -luuid -lversion -loleaut32 -lole32 -limm32 -lwinmm -lgdi32 -luser32 -lm  -Wl,--no-undefined -lmingw32 -lSDL2main -lSDL2 -mwindows 
```

![sdl-1-6-3](sdl\sdl-1-6-3.png)

这些依赖的外部库 gdi32，setupapi 可以 以静态的方式绑定，也可以以动态的方式引入。这是什么意思呢？如果 setupapi 动态引入，那 ffplay.exe 就会依赖 Setupapi.dll ，但是  libSDL2.a 的代码还是已经静态编译进去 ffplay.exe 了，所以 ffplay.exe 不依赖 SDL2.dll。

如果  setupapi 静态引入，那 ffplay.exe 不依赖 Setupapi.dll ，也不依赖  SDL2.dll。

但是，一个库 只能统一以静态方式引入，或者统一以动态方式引入。我说这句话是什么意思？例如 你静态引入了 setupapi 库，那其他模块就不能动态引入 setupapi 库。会报函数重定义，因为静态库里面有函数的实现，动态库里面也有函数的实现。

也可以使用 `sdl2-config --static-libs` 命令输出 SDL2 的静态库选项，`sdl2-config` 是 pacman 安装的一个脚本。如下：

![sdl-1-6-4](sdl\sdl-1-6-4.png)

`-Wl,--no-undefined` 这个选项的作用我也不太清楚，后续补充，反正这样填就可以编译通过了。

------

现在我们只需要让 FFmpeg 的编译脚本 能启用上面 这些 SDL2 的静态选项即可。

由于我没找到 FFmpeg 的 configure 脚本哪个参数能启用 这些静态选项，所以我选择直接改 configure 的脚本，改动内容如下：

```
sdl2_extralibs=$("${SDL2_CONFIG}" --libs)
```

替换成

```
sdl2_extralibs=$("${SDL2_CONFIG}" --static-libs)
```

再执行之前的 configure 指令，make 跟 make install，会可发现 ffplay.exe 不再依赖 SDL2.dll 动态库。因此上面的方法是成功的。

![sdl-1-6-5](sdl\sdl-1-6-5.png)

------

以上都是使用 MinGW 的 gcc 编译器来编译 FFmpeg，下面就来介绍一下如何使用 MSVC 编译器来编译FFmpeg，同时引入 MinGW 编译出来的 SDL2.dll 动态库。

提示：为防止前面的一些编译缓存影响操作，最好重新拉一下 FFmpeg-n4.4.1 的代码，我的习惯是保存一个原始的 zip 压缩包，每次要编译再重新解压出来。

![sdl-1-7](sdl\sdl-1-7.png)

------

下面开始用 MSVC 来编译FFmpeg，基本的操作跟之前的[《用msys2与msvc编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)是一样的。

在 MSVC 环境下，在 configure 的时候简单地加上 --enable-sdl2 是不行的。有兴趣可以试一下，根本编译不出来 ffpaly.exe。

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-4.4-msvc \
--enable-gpl \
--enable-nonfree \
--enable-shared \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--enable-sdl2 \
--toolchain=msvc
```

![sdl-1-8](sdl\sdl-1-8.png)

我们 pacman 安装 SDL 的时候，是有两个 文件一起安装的。libSDL2.dll.a 跟 SDL2.dll ，这两个文件，实际上就是 msvc 经常使用的 lib 导入库跟 dll 动态库。

虽然 它的名字后缀 是 .a，但实际上他跟 lib 导入库应该是一样的，这两个文件**应该**都能被 msvc 链接器使用的。

因此，**应该**只需要改动一下 ffmpeg 的编译逻辑即可。

我试了一下，发现改编译逻辑非常麻烦，MinGW 编译出来的动态库确实是可以被 MSVC 使用的，在 [《MinGW编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-shared.html)已经演示过了。这里没有搞定是因为 SDL 这个库本身他是依赖了很多东西。

------

最后我想了一个办法，用 MSVC 重新编译一下 SDL 的源码，这样编译生成的 导入库跟动态库应该可以被 msvc 使用。

先执行以下命令，删掉 之前 pacman 安装的 SDL，防止干扰。

```
pacman -R mingw-w64-x86_64-SDL2
```

选择 [release-2.0.18](https://github.com/libsdl-org/SDL/releases/tag/release-2.0.18) 版本下载，如下：

![sdl-1-9](sdl\sdl-1-9.png)



下载后可以发现，SDL官方其实已经建立好了 vs 的解决方案。

![sdl-2-1](sdl\sdl-2-1.png)

这样我们就不需要手动敲命令调 cl.exe 来编译 SDL 源码了，直接用 vs2019 打开这个工程，点几下按钮即可编译出 SDL 动态库，如下：

![sdl-2-2](sdl\sdl-2-2.png)

![sdl-2-3](sdl\sdl-2-3.png)

提示：SDL 有两个库都要编译，SDL2 跟 SDL2main 。SDL2main 是静态库。

------

到这里，我们已经用 vs2019 编译出 SDL 动态库了，现在只需要把这些动态库跟lib导入库，放到 msys2 的目录，让 FFmpeg 的编译脚本能找到 这些 lib 跟dll 即可。

先把 `SDL-release-2.0.18\include` 目录下的头文件全部拷贝到 `\usr\local\include\sdl2`，编译的时候要用。如下：

![sdl-2-4](sdl\sdl-2-4.png)

再把 `SDL2.lib` 跟 `SDL2main.lib` 拷贝到 `\usr\local\lib\x64` 目录，如下：

dll文件先不用管，dll只有运行候才会用到。

![sdl-2-5](sdl\sdl-2-5.png)

现在还需要在 `\usr\local\lib\pkgconfig` 目录下创建一个 sdl2.pc 文件，内容如下：

```
my_includedir=/usr/local/include/sdl2
libdir=/usr/local/lib/x64

Name: sdl
Description: sdl is very 666
Version: 2.0.18
Libs: -L${libdir} -lsdl2 -lSDL2main
Cflags: -I${my_includedir}
```

**创建 sdl2.pc 文件，是为了 pkg-config 命令能提取 sdl2.pc 的内容参数，传递给 cl.exe 编译器。**

要执行以下命令，把 `/usr/local/lib/pkgconfig/` 加入搜索路径：

```
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/:$PKG_CONFIG_PATH
```



------

现在就可以再次编译 FFmpeg 了，执行以下命令开始编译：

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-4.4-msvc \
--enable-gpl \
--enable-nonfree \
--enable-shared \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--enable-sdl2 \
--toolchain=msvc
```

![sdl-2-6](sdl\sdl-2-6.png)

看到 configure 通过的一刻，真是非常高兴。后续就是直接 make 跟 make install 就行。

------

编译成功后如下：

![sdl-2-7](sdl\sdl-2-7.png)

**现在只需要把 vs2019 生成的 sdl2.dll 复制到 ffplay.exe 同目录的地方，即可正常使用了。**

------

官方文档上描述虽然 SDL 可以编译为静态库，但不会主动介绍静态库的编译方法，也不推荐将 SDL 编译为静态库使用，所以暂时不讲解如何用 vs2019 编译出 SDL 的静态库。推荐阅读 [《SDL 静态编译教程》](https://meishizaolunzi.com/sdl-jing-tai-bian-yi/)

