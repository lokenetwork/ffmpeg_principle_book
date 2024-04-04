# 用Ubuntu18与clion调试FFmpeg—FFmpeg调试环境搭建

<div id="meta-description---">在FFmpeg 支持的众多系统中，Ubuntu 的调试环境是最容易搭建的，所以本书先讲 Ubuntu 的环境搭建。
如果是刚接触 FFmpeg，希望学习 FFmpeg 的 API 使用，我个人建议，直接使用 Ubuntu18 + clion 环境即可。因为 ffmpeg.c 的逻辑在所有平台都是一样的，选一个最容易搭建的平台来了解 ffmpeg.c 的逻辑即可，ffmpeg.c 的 5 千行代码里面有所有 API 函数的使用方法。</div>

FFmpeg 项目是由 C语言写的，C语言的跨平台性是比较差的，不同的平台有各自的线程API 以及一些各自的系统函数。因此如果用 C 语言来写一个项目，实现跨平台是比较麻烦的。

但是 FFmpeg 克服了这个问题，FFmpeg 主要通过一个 shell 脚本 （configure），在编译之前检测当前的系统环境，生成一个用于当前系统的 config.h 头文件。

config.h 头文件 定义了非常多的宏，根据这些宏，引入不同的平台函数跟代码，以此实现跨平台。

------

在FFmpeg 支持的众多系统中，Ubuntu 的调试环境是最容易搭建的，所以本书先讲 Ubuntu 的环境搭建。

如果是刚接触 FFmpeg，希望学习 FFmpeg 的 API 使用，我个人建议，直接使用 Ubuntu18 + clion 环境即可。因为 `ffmpeg.c` 的逻辑在所有平台都是一样的，选一个最容易搭建的平台来了解 `ffmpeg.c` 的逻辑即可，`ffmpeg.c` 的 5 千行代码里面有所有 API 函数的使用方法。

------

本文对应的视频地址：https://www.bilibili.com/video/BV1PV4y1Q7vo 

下面开始搭建，先下个 VMware ，然后安装上 Ubuntu18 的系统，安装完成后，如下：

![ubuntu18-1-1](.\ubuntu18\ubuntu18-1-1.png)

然后上 Github 下载 [FFmpeg-n4.4.1.zip](https://github.com/FFmpeg/FFmpeg/archive/refs/tags/n4.4.1.zip) 代码，我放到 Document 目录下面，如下：

![ubuntu18-1-2](.\ubuntu18\ubuntu18-1-2.png)



虽然 FFmpeg 是通过 `makefile` 编译的，但是还是可以用 **clion** 来调试，比 `gdb` 更直观一些。`clion2021` 的版本比较完善，早几年我用的时候，调试好像必须提供 `CMakeLists.txt` 文件，现在只有 `makefile` 文件也能用 **clion**  调试了。

先安装一些必要的软件：

```
# 安装以下软件。
apt-get install diffutils make pkg-config yasm
apt-get install libsdl2-2.0
apt-get install libsdl2-dev
```

开始执行 `configure` 脚本检测编译环境：

```
#创建目录
mkdir -p /home/ubuntu/ffmpeg/build64/

#进入 FFmpeg 源码目录
cd /home/ubuntu/Documents/FFmpeg-n4.4.1

#执行检测脚本
./configure \
--prefix=/home/ubuntu/ffmpeg/build64/ffmepg-4.4-ubuntu \
--enable-gpl \
--enable-nonfree \
--enable-debug=3 \
--disable-optimizations \
--disable-asm \
--disable-stripping

```

这里有三个重点要注意：

1，要编译出 `ffplay` 可执行文件，必须安装 `sdl2` 库

2，上面的 `configure` 不要开启动态库，静态库调试会方便很多。`ffmpeg 4.4.1` 你不加 `--enable-shared` 就是使用静态库编译。`configure` 的规则是静态库动态库只能二选一。

3，后面有好几个选项是开启 `debug` 模式，告诉编译器不要优化代码，因为有时候优化代码 会改变 代码原来的运行顺序，导致调试的时候跳转看起来很奇怪。

这是 编译器的优化技巧，改变代码原来的运行顺序是为了能让更多的指令能并行，最后运行的结果跟原来是一样的。指令级并行（Instruction-Level Parallelism）相关的技术可以看《深入理解计算机系统》第5章 跟 《计算机体系结构》第3章。

------

正式开始编译，执行以下命令，`-j` 参数可以指定编译线程数量，请选择合适的数量：

```
make -j16
```

到这里，FFmpeg 的编译就完成了，不需要执行 `make install`，因为不需要移动生成的二进制可执行文件到别的地方，我们直接用 `clion` 在项目目录进行调试。

现在 已经 编译生成 了 `ffmpeg` ，`ffplay` 可执行文件了，如下图：

![ubuntu18-1-3](.\ubuntu18\ubuntu18-1-3.png)

现在就可以用 clion 来调试这个 `ffmpeg_g` 文件了，clion 也是调 `gdb` 来做这个事情的。

提醒：后面有 `_g` 的代表有调试符号，没有 `_g` 的是进行了 `strip` 操作，把调试信息去掉了，所以要选择 _g 后缀的可执行文件来调试。要不会无法进行**断点调试**。

------

下载 **clion2021**，打开 FFmpeg-n4.4.1 目录，然后找到 根目录的 `Makefile` 文件，右键执行 "Load Makefile Project"，如下图：

注意：轻一点要执行一下 `clean` 操作，虽然 `clean` 会把之前编译出来的的 `ffmpeg_g`，`ffplay_g` 删掉，但你后面点击小蟑螂调试会再次编译出来 `ffmpeg_g`，`ffplay_g` 

提示2：如果没有  "Load Makefile Project" 选项可以选，请跳到文章后面查看解决方法。

如果 Load 失败，再执行多几次 Load 就可以了。 clion 有时候是有点抽风的。你 reload 多几次 makefile project。

![ubuntu18-1-4](.\ubuntu18\ubuntu18-1-4.png)

执行完之后，就会看到 很多 编译目标 （target），，如下图：

![ubuntu18-1-5](.\ubuntu18\ubuntu18-1-5.png)

上图中有很多 `target`，makefile 可以单独编译某个模块的`target`。但为了方便起见，我们选择 `all` ，全部编译。Makefile 有编译缓存，没修改的文件不会重新编译，所以 all 编译也没问题。

点击  "Edit Configurations " ，配置一些东西，如下：

![ubuntu18-1-6](.\ubuntu18\ubuntu18-1-6.png)

![ubuntu18-1-7](.\ubuntu18\ubuntu18-1-7.png)

walking-dead.mp4，下载地址：[网盘](/img/walking-dead.zip)，

------

然后 ffmpeg 这个可执行文件的源代码是 `fftools/ffmpeg.c` ，直接找到这个 文件，在 `main` 函数打一个断点，**点击右上角的小蟑螂**，即可调试 ffmpeg 的整个转封装 转码过程。如下图：

![ubuntu18-1-8](.\ubuntu18\ubuntu18-1-8.png)

------

下面讲一下 如何调试 ffplay 播放器，也是只需要改动一下 "Edit Configurations " 即可，如下：

![ubuntu18-1-9](.\ubuntu18\ubuntu18-1-9.png)

ffplay 播放器的源码文件是 `fftools/ffplay.c` ，也是在 main 函数打个断点即可调试播放器的流程，如下图：

![ubuntu18-2-1](.\ubuntu18\ubuntu18-2-1.png)

------

到这里 Ubuntu18 + clion2021 调试 FFmpeg 已经讲解完毕，最后补充一些 clion 的使用技巧。

上面的搭建过程，我们是右键用的  **Load Makefile Project**，有时候 clion 会出点问题，没有这个 "Load Makefile Project" 给你选，这种情况，你需要自己添加。

Makefile 类型的项目 在 clion 里面有两种 configuration （配置）：**Makefile Target** 跟 **Makefile Application** 。

可以这么理解  Makefile Target ，他就跟 在命令行 执行 make xxx 一样，xxx 是 target，不填就是默认的，ffmpeg 默认的 target 是all。

Makefile Application 这个功能，就类似 在 命令行 执行 `gdb ./ffplay` 一样。Makefile Application 也是 clion 调 gdb 来调试 Makefile Target 编译生成的文件。

因此 如果  clion 出问题，没有这个 "Load Makefile Project" 给你选，你可以直接添加 Makefile Application，把下面这些信息填好，如下：

![ubuntu18-2-2](.\ubuntu18\ubuntu18-2-2.png)

具体的操作步骤请看 B 站视频，https://www.bilibili.com/video/BV1PV4y1Q7vo 

------

clion 调试的时候有一个 [watch point](https://www.jetbrains.com/help/clion/using-watchpoints.html#use-watchpoints) 功能，也叫做 data breakpoint （数据断点），通过这个功能，你可以快速定位到哪些代码修改了某个变量。

例如我知道 `avcodec_alloc_context3` 会申请一个编码器内存，但是我不知道什么哪里设置了  `pix_fmt` ，`width`，`height` 之类的编码器信息，那我就可以在 `avcodec_alloc_context3` 的时候设置行数断点，然后 再针对 `pix_fmt` 设置一个 watch point ，如下：

![ubuntu18-2-3](ubuntu18\ubuntu18-2-3.png)

![ubuntu18-2-4](ubuntu18\ubuntu18-2-4.png)

生命周期请选择 Persistence，如下：

![ubuntu18-2-4](ubuntu18\ubuntu18-2-5.png)

设置好 watch point 之后，让代码继续跑，碰到 修改 pix_fmt 的地方就会停下，如下：

![ubuntu18-2-6](ubuntu18\ubuntu18-2-6.png)

从上面的函数调用栈，可以看到，是在 `init_output_stream_encode` 函数里面设置 编码器的 pix_fmt 的
