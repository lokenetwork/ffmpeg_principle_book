# ShiftMediaProject项目介绍—FFmpeg调试环境搭建

<div id="meta-description---">ShiftMediaProject项目介绍</div>

前面用 WinDgb 跟 VsDebug 断点调试 FFmpeg 源码还不太方便，有没有一种方法能跟 Ubuntu + clion 那样直观地看到各种数据结构 ？

这里就需要介绍一下 **Matthew** 维护的 ShiftMediaProject 这个项目。虽然这不是一个官方的项目，但是非常好用。

截图一下项目介绍，如下：

![smp-1-1](smp1\smp-1-1.png)

ShiftMediaProject 下面有很多子项目，包括 FFmpeg，x264，SDL 等等。**虽然 FFmpeg 跟 x264 本身就支持 MSVC 的方式编译，他们的 configure 跟 makefile 脚本都可以使用 msvc 的编译器的**。但是那始终是命令行工具，不太方便。

所以 Matthew 把这些编译脚本，移植成了 Visual Studio 原生的解决方案（solution）。这样点几个按钮就可以编译了，而且可以断点调试源码，查看数据。

VS 的版本众多，目前 vs2013 ~ vs2019 都可以使用 ShiftMediaProject 。

------

在 [《快速上手vs2019》](https://ffmpeg.xianwaizhiyin.net/base-compile/vs2019.html) 讲过， vs2019 实际上也是调 cl.exe 跟 lib.exe， link.exe 完成编译链接。

------

因为 ShiftMediaProject 下面有很多个 音视频项目，我们先看 FFmpeg 这个项目，下载 [FFmpeg](https://github.com/ShiftMediaProject/FFmpeg/releases?page=2)，如下：

![smp-1-1](smp1\smp-1-2.png)

从上图可以看到，其实你可以直接用他已经编译好的 lib 跟 dll，上面是已经编译好的各个 msvc 版本的 lib 跟 dll。但是我们还是下载个 Source code 源码压缩包，看一下里面的内容。

ShiftMediaProject 会把 FFmpeg 官方的每一次版本发布都移植 成 vs 原生的解决方案，上图的那个 r100605 就是 FFmpeg 官方的 版本，ShiftMediaProject 的源码是跟 FFmpeg 官方保持一致的，移植的只是编译方式。

------

我习惯把 ShiftMediaProject 的东西放到 D 盘 的 shift-media-project  目录， shift-media-project  是我自己的目录。

ShiftMediaProject 项目建议，要创建两个文件夹 msvc 跟 source，如下：

![smp-1-1](smp1\smp-1-2-1.png)

msvc 是输出文件夹，vs2019 编译输出的文件都在这里，然后 source 目录下存放 ShiftMediaProject 的各个子项目的源码，例如 FFmpeg，x264，SDL 等等。

下载 ShiftMediaProject  的 4.4.r100605 的 FFmpeg 源码之后，放到 D 盘 的 source 目录，如下：

![smp-1-1](smp1\smp-1-3.png)

在 SMP 下有个 readme.txt ，介绍了项目如何配置。**ShiftMediaProject  的 FFmpeg 依赖一些外部的项目**，如果你现在直接打开 FFmpeg 下面的 SMP 里面的 sln 文件，是无法编译通过的。

Shift Media Project 下面的大部分每个音视频项目x264，ffmpeg，libssh ，gnutls 等等的 SMP 目录下面都有一个 `project_get_dependencies.bat` 脚本，这个脚本是他解决依赖的脚本。

下图 可以看到 FFmpeg 需要哪些扩展库，这些库都是需要安装编译的。Shift Media Project 把所有 的编码器都集成在里面，不想要 x265 都不行，好像都没有选项配置，要手动改。

补充：好像 bat 脚本有个 PASSDEPENDENCIES 参数。这里埋个坑，后面研究。

我不打算用 这个 bat 脚本来解决依赖。因为不太好控制版本。而且 git 没有科学上网会很慢，我们也不需要查看 git log ，所以可以直接下他的 release 源码压缩包 来手动编译。依赖脚本只是用来参考他的编译思路。

如果跑 FFmpeg 这个 bat 脚本来解决依赖，网络不好可能要 几个小时。

![smp-1-3-1](smp1\smp-1-3-1.png)

上面的 FFmpeg 依赖的库，都能在  Shift Media Project 里面搜索到，如下：

**提示：最好用 搜索框搜索 Shift Media Project 里面的子项目，因为他有分页，第一页可能没有 gnutls。**

![shift-media-project-1-3-2-1](shift-media-project\shift-media-project-1-3-2-1.png)

我们再把 [gnutls](https://github.com/ShiftMediaProject/gnutls/releases/tag/3.7.5) 项目下载来看一下，如下：

![smp-1-3-2-2](smp1\smp-1-3-2-2.png)

会发现 gnutls 的 smp 下面也有一个 project_get_dependencies.bat 脚本，内容如下：

![smp-1-3-2-3](smp1\smp-1-3-2-3.png)

从上图可以看出，gnutls 项目依赖 nettle ，gmp 跟 zlib 项目。

所以 这个 project_get_dependencies.bat 脚本是一个嵌套关联项目的脚本，例如 FFmpeg 依赖 gnutls，然后 gnutls 依赖 nettle ，gmp 跟 zlib。

FFmpeg 这个 bat 脚本会不断的把依赖项目下载完毕。

了解了 bat 脚本的逻辑，我们就不需要使用这个脚本了，可以自己手动自定义 FFmpeg，把不需要的扩展库删掉，节省下载时间。

下篇文章讲具体操作。

------

参考资料：

1，[《vs2019编译ffmpeg源码为静态库动态库》](https://blog.csdn.net/yao_hou/article/details/121581878)



