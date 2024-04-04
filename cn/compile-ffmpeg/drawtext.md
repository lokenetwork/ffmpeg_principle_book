# FFmpeg引入文字水印扩展—FFmpeg编译教程

<div id="meta-description---">给视频加上时间戳水印是一个比较常见的需求，FFmpeg 提供了一个 drawtext 滤镜来给视频加上文字水印</div>

FFmpeg 提供了一个 **drawtext** 滤镜来给视频加上文字水印，通常的应用场景就是给监控视频打上日期时间戳。

`drawtext` 滤镜依赖 [freetype](https://freetype.org/) 库，所以在编译 FFmpeg 之前，需要安装好 `freetype` 库。 **`freetype` 库负责把文字转成图片**，然后叠加到视频上去。

还有另外两个库也可以用在 `drawtext` 滤镜里，分别是 [fontconfig](https://www.freedesktop.org/wiki/Software/fontconfig/)，[fribidi](https://github.com/fribidi/fribidi)。这两个库不是必须的，即便你没安装这两个库，也能使用 `drawtext` 滤镜，只是会少一些功能。

`fontconfig` 是一个配置管理字体文件的库，如果你的电脑安装了成千上万的字体，在使用 `ffmpeg.exe` 添加文字水印的时候，可以更快找到指定的字体，当然，这个库还有一些其他的功能，推荐阅读他们的官网介绍。

`fribidi` 库的 bidi 是 Bidirectional Algorithm（双向算法）的缩写。`fribidi` 库是用来处理双向文字的，Unicode 编码支持非常多的语言，虽然我们常用的中文，英文是从左到右读的。但是有一些语言是从右到左读的，例如 阿拉伯文 ，希伯来文。

如果没用到 阿拉伯文 ，希伯来文，那就可以不安装 `fribidi`， `drawtext` 滤镜也可以正常打上中文英文的时间戳水印。

不过本文不讲解 `fontconfig`，`fribidi` 库的安装，因为对于 `drawtext` 滤镜来说，这两个库不是必须的。

本文主要讲解如何在 `mingw`，`msvc` 的环境下安装 `freetype` 库，以及如何编译出带 `drawtext` 的 `ffmepg.exe`

----

#### 在 mingw 环境下安装 `freetype` 库

`mingw` 环境下安装 `freetype` 库的过程非常简单，参考《[FFmpeg引入SDL扩展](https://ffmpeg.xianwaizhiyin.net/compile-ffmpeg/sdl.html)》一文，由于 msys2 有云仓库，可以直接通过 `pacman` 命令直接安装 `freetype` 库到本地，不用自己下源代码进行编译。

我也通常不知道一个包的准确名字，我是通过去 云仓库的网页上 搜索的，如下：

![1-1](drawtext\1-1.png)

![1-2](drawtext\1-2.png)

下载安装 `freetype` 库 的命令如下：

```
pacman -S mingw-w64-x86_64-freetype
```

![1-3](drawtext\1-3.png)

安装完成之后，我们可以通过下面的命令，查看 `pacman` 命令安装了哪些东西到我们的电脑，如下：

```
pacman -QL mingw-w64-x86_64-freetype
```

![1-4](drawtext\1-4.png)

上图中的 `libfreetype.a` 是静态库，`libfreetype.dll.a` 是动态库的导入库，链接阶段用的。

而 `freetype2.pc` 里面定义了使用 `freetype` 库的编译器，以及链接器的 选项。`pkg-config` 命令可以把 pc 文件里面的信息提取出来。

----

安装完 `freetype` 库之后，我们就可以正式编译 FFmpeg，这里参考《[用msys2与mingw编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw.html)》，只需要在 `configure` 的时候开启使用 `freetype` 即可，如下：

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
--enable-shared \
--enable-libfreetype
```

上面的命令跑完之后，可以看到输出的 Enabled filters 里面已经有了 `drawtext` 滤镜了，如下：

![1-5](drawtext\1-5.png)

提示：`drawbox`，`drawgraph`，`drawgrid`，也是比较有趣的滤镜，可以试玩一下。

编译安装完成之后，需要再下载一个字体文件 [CascadiaCode.ttf](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/other_data/CascadiaCode.ttf)，放到跟 `ffmpeg.exe`  同级的目录，如下：

![1-6](drawtext\1-6.png)

下面就可以运行 `ffmpeg.exe` 进行测试，加时间戳水印是否成功，命令如下：

```
cd /home/loken/ffmpeg/build64/ffmpeg-n4.4.1-mingw/bin
```

```
./ffmpeg.exe -i juren.mp4 \
-vf "drawtext=fontfile=CascadiaCode.ttf:text='2023-02-13-':timecode='18\:51\:05\:00':r=60:x=10:y=10:fontsize=80:fontcolor=white" \
-an -t 10 -f mp4 output.mp4 -y
```

![1-7](drawtext\1-7.png)

上面的滤镜语法逻辑解释如下：

1. `fontfile=CascadiaCode.ttf`，指定需要使用的字体文件。
2. `text='2023-02-13-'`，指定水印文字，这是固定的文字。
3. `timecode='18\:51\:05\:00'`，指定起始时间，会在这个时间上不断递增，这是动态的文字。
4. `r=60`，r 的全称是 rate，速率的意思，也就是跑够 60 就会加 1，如果是 2 倍速播放视频，可以把这个 r 设置成 30，这样跑够 30 就会加

效果图如下：

![1-8](drawtext\1-8.png)

读者可以通过下面命令查询 `drawtext` 滤镜 的所有参数，如下：

```
ffmpeg -hide_banner -h filter=drawtext
```

![1-9](drawtext\1-9.png)

----

#### 在 msvc 环境下安装 `freetype` 库

操作之前，先参考《[用msys2与msvc编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)》文章，进入 msvc 的 msys2 环境。

在 `msys2` + `msvc` 环境下编译 `ffmpeg` + `freetype` 稍微有点麻烦，因为不能直接使用 `pacman` 命令安装。

不过也不算复杂，我们自己创建一个 `freetype2.pc` 文件，然后下载好 `msvc` 的 `freetype` 库，放在对应的目录下就行，实际上就是手动干了 `pacman` 命令的活。

`freetype` 库的 `msvc` 版本可以在 [GitHub](https://github.com/ShiftMediaProject/freetype2) 下载，不用自己编译 源代码。本文选用的是 [2-11-0-1-msvc16](https://github.com/ShiftMediaProject/freetype2/releases/tag/VER-2-11-0-1) 版本，msvc16 版本对应 Visual Studio 2019

下载解压之后，目录如下：

![2-1](drawtext\2-1.png)

![2-2](drawtext\2-2.png)

![2-3](drawtext\2-3.png)

可以看到，只有静态库，动态库，但是没有 `pkg-config` 命令需要的那个 `freetype2.pc` 文件，所以我们需要自己写 pc 文件，可以直接借鉴 `mingw` 的 `freetype2.pc` 文件，修改后的内容如下：

```
prefix=/usr/local
exec_prefix=/usr/local
libdir=/usr/local/lib/x64
includedir=/usr/local/include

Name: FreeType 2
URL: https://freetype.org
Description: A free, high-quality, and portable font engine.
Version: 21.1.01
Requires:
Requires.private: zlib
Libs: -L${libdir} -lfreetype
Libs.private:
Cflags: -I${includedir}/freetype2
```

我们把新创建的 `freetype2.pc` 移动到 `/usr/local/lib/pkgconfig` 目录，如下：

![2-4](drawtext\2-4.png)

再执行下面的命令，把 `/usr/local/lib/pkgconfig` 目录，加进去 `pkg-config` 命令的搜索路径。

```
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/:$PKG_CONFIG_PATH
```

然后我们需要把 `freetype` 相关的头文件 与 库，拷贝到 `freetype2.pc` 文件里面定义好的目录，如下：

![2-6](drawtext\2-6.png)

![2-7](drawtext\2-7.png)

这些准备工作都完成之后，我们就可以编译 FFmpeg 了，基本跟《[用msys2与msvc编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)》一文类似，只是需要在 `configure` 的时候开启使用 `freetype`，如下：

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-4.4-msvc-freetype \
--enable-gpl \
--enable-nonfree \
--enable-shared \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--toolchain=msvc \
--enable-libfreetype
```

编译完成之后，好像不需要把 `freetype.dll` 复制到 `ffmpeg.exe` 的同级目录，也能跑起来，我用 WinDbg 的 `lmf` 命令查看了一下，发现加载了 JDK 里面的  `freetype.dll`，如下：

![2-8](drawtext\2-8.png)

加载的  `freetype.dll` 并不是我们预期的 dll，所以还是需要 `freetype.dll` 复制到 `ffmpeg.exe` 的同级目录，然后就可以愉快地加水印了，命令如下：

```
.\ffmpeg.exe -i juren.mp4 ^
-vf "drawtext=fontfile=CascadiaCode.ttf:text='2023-02-13-':timecode='18\:51\:05\:00':r=60:x=10:y=10:fontsize=80:fontcolor=white" ^
-an -t 10 -f mp4 output.mp4 -y
```

![2-9](drawtext\2-9.png)

---

参考资料：

1，《[BiDi 算法详解及应用](https://blog.csdn.net/spring1208/article/details/52444215)》

