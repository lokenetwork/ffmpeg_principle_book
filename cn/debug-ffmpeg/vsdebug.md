# 用VsDebug断点调试FFmpeg—FFmpeg调试环境搭建

<div id="meta-description---">前面两篇文章已经讲解了如何在 windows 编译出 ffmpeg.exe 文件。
在 Windows 平台有没类似  gdb 调试工具可以断点调试可执行文件呢？
Windows 平台主要有两款调试工具。
1，VsDebug，集成在 vs2019 里面的调试器。vs 系列都是用的 VsDebug。
2，WinDbg，Windows诞生之初的第一款功能全面的调试器，目前依然应用广泛。例如 qt creator 的 MSVC 环境调试就是用的 WinDbg。</div>

《[用msys2与mingw编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw.html)》[《用msys2与msvc编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html) ，前面两篇文章已经讲解了如何在 windows 编译出 ffmpeg.exe 文件。

在 Windows 平台有没类似  gdb 调试工具可以断点调试可执行文件呢？

Windows 平台主要有两款调试工具。

1，VsDebug，集成在 vs2019 里面的调试器。vs 系列都是用的 VsDebug。

2，WinDbg，Windows诞生之初的第一款功能全面的调试器，目前依然应用广泛。例如 qt creator 的 MSVC 环境调试就是用的 WinDbg。



------

VsDebug 是 vs2019 的调试器，架构图如下：

![vsdebug-1-1](vsdebug\vsdebug-1-1.png)

上图中 devenv.exe 就是 vs2019 的界面程序。

用 vs2019 打开一个简单的 C 程序项目，例如之前的 hello.exe ，点击菜单栏的调试按钮，再查看相关的进行，就会看到 msvsmon.exe 进程，这个进程只有开启调试才会出现，编辑代码不会出现。

![vsdebug-1-2](vsdebug\vsdebug-1-2.png)

![vsdebug-1-3](vsdebug\vsdebug-1-3.png)

上图为什么叫 Remote Debugger ，因为 VsDebug 最初是为远程调试设计的，所以VS 中一直把他称为远程调试器。

------

下面就用 vs2019 来调试一下之前 编译出来的 `ffmpeg.exe` ，用的是 `msvc` 编译出来的 `ffmpeg.exe`。没用 `mingw` 的 `ffmpeg.exe` 是因为他依赖一些外部库，操作会比较麻烦。

**提醒：**编译成功之后千万不要删除或者移动 FFmpeg-n4.4.1 源码目录，因为符号对应的源代码位置写进去了。移动了目录会导致调试的时候就会找不到源代码。

现在用 vs2019 打开文件夹 `ffmpeg\build64\ffmepg-4.4-msvc` ，如下：

![vs2019-1-1](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-1.png)

![vs2019-1-2](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-2.png)

![vs2019-1-3](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-3.png)

由于我在 `configure` 的时候没有指定 `SDL.dll` 的位置，所以 ffplay.exe 没有编译出来，这个读者自行探索。

------

上面的流程，我只是用 vs2019 打开了一个文件夹，没有创建 解决方案 （solution），这是因为 在 Visual Studio 2017 及更高版本中 ，可以不创建解决方案，直接调试 `exe` 文件。

不创建解决方案直接调试 `exe` ，依赖两个文件，如下：

1，`tasks.vs.json` ，这个文件 类似 clion 的 Makefile Target，就是 make ，make install，make clean。本文是在 msys2 命令行手动编译 ffmpeg 的，所以不用配置 tasks.vs.json。详情请看[《tasks.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/tasks-vs-json-schema-reference-cpp?view=msvc-170)

2，`launch.vs.json`，这个文件 类似 clion 的 Makefile Application，配置要调试的可执行文件，参数之类的，这个文件是本文重点。

如何创建 `launch.vs.json` 文件呢？首先，右边点击 ffmpeg.exe 文件，就会显示 "Debug and Launch Setting"，如下

![vs2019-1-4](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-4.png)

因为是 MSVC 编译的 ffmpeg ，所以选择 Native 原生的方式，如下：

![vs2019-1-5](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-5.png)

这样 `launch.vs.json` 就创建完成了，这个文件是在 .vs 隐藏目录下的，如下：

![vs2019-1-6](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-6.png)

再 把 ffmpeg.exe 设置为启动调试选项，如下：

![vs2019-1-7](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-7.png)

然后，菜单栏的绿色按钮就可以点击了。如下：

![vs2019-1-8](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-8.png)

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

如上，我创建了两个调试选项，一个是 打印 help信息，一个是转码 `walking-dead.mp4` ，这个 mp4 文件请下载放到 `ffmepg-4.4-msvc` 目录下面。

更多 `launch.vs.json` 参数请查看微软的文档[《launch.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/launch-vs-schema-reference-cpp?view=msvc-170)。

上面的配置，因为 windows 的编解码跟 Linux 有点不一样，要指定一些具体的采样率信息，所以我直接 -c copy 不解码了。

配置之后，可以看到，菜单栏有两个调试选项可以选择，如下：

![vs2019-1-9](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-9.png)

咱们现在选择 ffmpeg.exe mux 来进行调试。如果直接点击 绿色三角按钮，会直接运行完毕，默认是没有断点的，所以我们需要先添加一个断点。

添加断点其实 跟 gdb 的 b main 差不多。点击菜单栏 的 Debug → Windows → Breakpoits ，即可添加断点。如下：

![vs2019-1-10](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-10.png)

现在再点绿色三角符号，就可以直接停在 main 函数里面，如下：

![vs2019-1-11](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-11.png)

现在，你可以在 `ffmpeg.c` 的源文件的具体行数打断点，方便调试。

由于我们没有在 vs2019 建立解决方案，是直接用 VsDebug 来调试，所以代码函数没有高亮，变量右侧也没有实时提示，只有底部一个 watch 区域可能看到变量数据。

跟之前在 ubuntu18 clion 的环境有点差距，`clion` 里面变量右侧是有实时信息的。如下：

![vs2019-1-12](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-12.png)

vs2019 其实也是可以建立解决方案，github 有个开源项目 [ShiftMediaProject](https://github.com/ShiftMediaProject/FFmpeg/tree/master/SMP)。VsDebug 跟 watch 在一般情况够用。

------

vs2019 也可以直接 调 VsDebug 直接附加到现有进程上进行调试，跟 GDB 附加进程一样。

点击 Debug → Attach to Process， 如下：

![vs2019-1-13](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/windows10-vs2019/vs2019-1-13.png)

参考资料：

1，[《自定义“打开文件夹”开发的生成和调试任务》](https://docs.microsoft.com/zh-cn/visualstudio/ide/customize-build-and-debug-tasks-in-visual-studio?view=vs-2022)

2，[《launch.vs.json 架构参考》](https://docs.microsoft.com/zh-cn/cpp/build/launch-vs-schema-reference-cpp?view=msvc-170)
