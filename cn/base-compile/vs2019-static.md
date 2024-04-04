# vs2019使用静态库—编译链接基础知识

<div id="meta-description---">在前文 《MSVC编译静态库》已经讲解了，如何使用 vs2019 编译出静态库。但是当时的静态库是直接在 命令行敲 link.exe 命令来使用的。如果我们想在 vs2019 里面使用这个静态库，应该怎么操作？这就是本文的主要内容。</div>

在前文 [《MSVC编译静态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc-static.html)已经讲解了，如何使用 vs2019 **编译出静态库**。但是当时的静态库是直接在 命令行敲 link.exe 命令来使用的。

如果我们想在 vs2019 里面使用这个静态库，应该怎么操作？这就是本文的主要内容。

用 vs2019 新建一个 **空白项目**，命名为 **zeus-vs** ，如下：

![vs2019-static-1-1](vs2019-static\vs2019-static-1-1.png)

然后记得设置 Platform 为 64 位，因为静态库是 64 位的。

然后添加 zeus.c 文件，如下：

![vs2019-static-1-2](vs2019-static\vs2019-static-1-2.png)

![vs2019-static-1-3](vs2019-static\vs2019-static-1-3.png)

其他 3 个头文件 sun.h 等也要加进去项目。



------

现在开始在 vs2019 配置 zeus 依赖的静态库。静态库是在 链接阶段的 用的，所以在 Linker 那里找配置就行，肯定在那里。

果然，添加静态库在 Linker ➜ Input ➜ Additional Dependencies 里面。

![vs2019-static-1-4](vs2019-static\vs2019-static-1-4.png)

虽然配置了 静态库名称，但是静态库的搜索路径还未加上去，所以还要修一下配置，如下：

![vs2019-static-1-5](vs2019-static\vs2019-static-1-5.png)

注意看底部的提示，你每点中一个配置，他对应的 link.exe 命令行参数都会提示出来。

我们这种是通过界面改 Linker 的配置来使用引用静态库，但是 微软有更集成的方法，请看 [《演练：创建并使用静态库》](https://docs.microsoft.com/zh-cn/cpp/build/walkthrough-creating-and-using-a-static-library-cpp?view=msvc-170)

**但是无论哪种方式，最后都是 往 link.exe 命令加个参数。**

------

然后 点击菜单栏的 build ，就会生成 zeus-vs.exe 文件了，如下：

![vs2019-static-1-6](vs2019-static\vs2019-static-1-6.png)

