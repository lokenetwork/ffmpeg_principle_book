# vs2019使用动态库—编译链接基础知识

<div id="meta-description---">在前文 《MSVC编译动态库》已经讲解了，如何使用 vs2019 **编译出动态库**。但是当时的动态库是直接在 命令行敲 link.exe 命令来使用的。如果我们想在 vs2019 里面使用这个动态库，应该怎么操作？这就是本文的主要内容。</div>

在前文 [《MSVC编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc-shared.html)已经讲解了，如何使用 vs2019 **编译出动态库**。但是当时的动态库是直接在 命令行敲 link.exe 命令来使用的。

如果我们想在 vs2019 里面使用这个动态库，应该怎么操作？这就是本文的主要内容。

用 vs2019 新建一个 **空白项目**，命名为 **zeus-vs2** ，注意是 **zeus-vs2**，如下：

![vs2019-shared-1-1](vs2019-shared\vs2019-shared-1-1.png)

然后记得设置 Platform 为 64 位，因为动态库是 64 位的。

然后添加 zeus.c 文件，如下：

![vs2019-static-1-2](vs2019-static\vs2019-static-1-2.png)

![vs2019-static-1-3](vs2019-static\vs2019-static-1-3.png)

其他 3 个头文件 sun.h 等也要加进去项目。



------

实际上，**使用动态库的编译链接步奏跟 使用静态库是一样的**，只不过动态库用的是 star.lib （导入库）。dll 在运行时才需要。

跟 使用静态库的操作一样，修改以下配置 Linker ➜ Input ➜ Additional Dependencies 

![vs2019-shared-1-2](vs2019-shared\vs2019-shared-1-2.png)

要再加上 star.lib 导入库的搜索路径，如下：

![vs2019-shared-1-3](vs2019-shared\vs2019-shared-1-3.png)

然后 点击菜单栏的 build ，就会生成 zeus-vs2.exe 文件了，如下：

![vs2019-shared-1-4](vs2019-shared\vs2019-shared-1-4.png)

但是运行的时候会报错，因为 star.dll 没有拷贝到 相同目录，如下:

![vs2019-shared-1-5](vs2019-shared\vs2019-shared-1-5.png)

这个问题，只需要把 star.dll 拷贝过来就完事了，运行的时候需要 star.dll

![vs2019-shared-1-6](vs2019-shared\vs2019-shared-1-6.png)
