# FFmpeg编译参数分析—FFFmpeg configure源码分析

**FFmpeg** 的编译参数是通过 **configure** 脚本来提供的，configure 可以接受各种编译参数，生成 `config.mak` 来传递给 `makefile` ，还会生成 `config.h` 给 C 程序 include 引入。

由于 configure 脚本的编译参数是非常多的，本文主要讲解一些比较常用的编译参数，一些特殊的编译参数，读者可通过以下命令查询。

```
configure --help
```

------

**1，**`--logfile=FILE` ，指定环境检测日志文件，默认是 `ffbuild/config.log`，`configure` 检测当前的环境能不能正常编译 FFmpeg 源代码的方法，就是实际编译一个函数，一小段代码。**logfile** 实际上就是 configure 脚本的运行日志。

**2，**指定各种安装目录，`--prefix`，`--bindir`，`--libdir`，`--shlibdir`，`--incdir`，`--pkgconfigdir`，如下：

![1-1](configure-args\1-1.png)



`prefix` 代表前缀目录，`libdir` 代表静态库目录，`shlibdir` 是动态库的安装目录，默认会把动态库安装到系统的动态库目录，也就是 `LIBDIR` 变量。

`pkgconfigdir` 代表 `pkg` 文件的安装目录，`pkg` 是用来给第三方软件找到 FFmpeg 静态库，动态库的安装目录的。

------

**3，**`--disable-static`，`--enable-shared`，这两个选项分别是 禁止生成静态库 跟 生成动态库。FFmpeg 默认会生成静态库，但是他不能同时生成静态库跟动态库，如果你启动了 `--enable-shared`，**那就只会生成动态库，不会生成静态库。**如果你需要同时用到静态库跟动态库，那就需要编译两次

**4，**`--enable-small`，把 FFmpeg 的体积减少。

**5，**`--disable-programs`，不生成 `ffmpeg.exe` ，`ffplay.exe` ，`ffprobe.exe` 可执行文件。只生成静态库或者动态库。也可以单独指定某个 exe 不生成，例如 `--disable-ffplay`

**6，**`--disable-doc`，不编译生成文档，可以节省编译时间。

**7，**`--disable-avdevice`，`--disable-avcodec`，`--disable-avformat`，`--disable-swresample`，`--disable-swscale`，`--disable-postproc`，`--disable-avfilter`。如果你只需要exe文件，可以指定不生成这些库，但是可能 exe 会缺少某个库的功能，具体待确认。

------

**8，第8条是非常重要的，代表是否关闭多线程**，`--disable-pthreads`，`--disable-w32threads`，`--disable-os2threads`。

其中 `pthread` 是 Linux 的 线程 API，`w32thread` 是 Windows 的线程API，`os2threads` 是苹果系统的线程 API。

我们来看一下，如果启用 `--disable-pthreads`，会影响哪些地方。`configure` 脚本处理 `--disable-pthreads` 这个选项的代码在 4088 行，如下：

![1-2](configure-args\1-2.png)

![1-3](configure-args\1-3.png)

由于 `threads` 是在 `$CMDLINE_SELECT` 里面的，所以会跑进去第二个条件。

所以 `--disable-pthreads` 这个选项的作用，实际上就是在 shell 里面设置一个变量 `pthreads = no`，我们接下来再看看 `pthreads`  这个变量会影响后续的哪些编译操作。

PS：我个人感觉 shell 写的 `configure` 非常难懂，不易调试，一样的环境检测功能，完全可以用 python 来写，更容易维护一些。

![1-4](configure-args\1-4.png)

上面的代码，是如果启用 `pthreads` 但是 没启用 `w32thread` 跟 `os2thread` 就会跑进去的逻辑

从上图可以看到 `pthreads` 等于 yes 的时候，就会 加上 `-pthreads` 选项给**链接器**，我们设置成 no，就不会跑进去上面的逻辑，也就没有 `-pthreads ` 选项。

`pthreads` 影响的地方就是这些了，但是写过 Linux 多线程的都知道，这只是编译的时候不链接线程库，代码里面还是有多线程代码，如果不隐藏多线程代码，链接的时候就会报错。

下面我们再来看一下，`ffmpeg.exe` 的多线程代码在哪里，又是如何隐藏的。

首先，`configure` 脚本会生成 `config.h` 文件，如下：

![1-5](configure-args\1-5.png)

`config.h` 的内容如下：

![1-6](configure-args\1-6.png)

从上图可以看到，定义了各种宏，C源代码里面正是引入了 `config.h` ，**利用宏来判断要不要启用 多线程的代码的**，现在只需要搜索 `HAVE_PTHREADS` 这个宏，即可找到多线程代码的位置，如下：

![1-7](configure-args\1-7.png)

可以看到，就在 `slicethread.c` 里面。

最后，提醒一下，configure 默认是自动判断当前环境是否支持多线程，如果支持就会自动开启。大部分系统都有多线程函数，所以可以认为默认是启用多线程的。多线程用于编解码模块，**如果禁用了，性能会下降非常多。**

------

**9，**`--disable-network`，如果不需要处理网络协议，可以启动这个选项，可以减小软件大小 跟 节省编译时间。

**10，**configure 脚本提供了各种裁剪功能，例如可以只启用某个编解码器，其他的编解码器全部不要，这样能大大缩小可执行文件的体积。

做法是先用 `--disable-encoders` 禁用所有的编码器，然后用 `--enable-encoder=NAME`启用某一个编码器。

封装格式也可以这样裁剪，`--disable-muxers` 禁用所有的复用器，`--enable-muxer=NAME` 启用某一个复用器。

由于 默认的编译会给 ffmpeg.exe 加上很多的编/解码器跟解/复用器，在嵌入式设备上为了使程序体积更小，可以采用此种方法。

其他的 滤镜，协议，也可以如此裁剪。

**11，**`--enable-libx264`，启用 `x264` 作为 `h.264` 的编解码器。

**12，**`--enable-libx265`，启用 `x265` 作为 `h.265` 的编解码器。

------

**13，**`--cc=CC` 指定 C 程序的编译器。

**14，**`--cxx=CXX`  指定 C++ 的编译器。

**15，**`--ld=LD` 指定链接器。

**16，**`--extra-cflags`，传递 标识选项 给 C 编译器。

**17，**`--extra-cxxflags`，传递 标识选项 给 C++ 编译器。

**18，** `--extra-ldflags`，传递 标识选项 给 LD 链接器。

**19，** `--extra-ldexeflags`，生成 exe 的时候传递给 链接器 的 选项。

**20，** `  --extra-ldsoflags`，生成 so 动态库的时候传递给 链接器 的 选项。

**21，**`--extra-libs`，指定额外的库，实际上就是往链接器加 选项。

------

**22，**`--env="ENV=override"` 这个是最重要的，可以覆盖环境变量。

**23，**`--custom-allocator`，自定义内存分配器，可以把 `malloc` 换成`jemalloc` 之类的。

------

写在最后，所有的选项 在 `configure --help` 里面都可以看到，而且有一些注释讲解，比较容易看懂。

![1-8](configure-args\1-8.png)
