# Android APP的构建过程

<div id="meta-description---">本文主要讲解 Android Studio 集成开发环境 是怎么把项目的代码编译，链接，然后打包成 apk。这些知识相当重要，在排查各种编译问题的时候非常有用。</div>

在阅读本文之前，请先参考《Adroid 第一行代码》第三版第一章的内容，安装好 Android 的开发环境，并且搭建好一个 Hello Word 的例子。

本文不会讲解 Android 项目的目录结构 以及 各个文件的作用，等等这些基础知识，因为《Adroid 第一行代码》里面已经讲了，如下：

<img src="https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/android_build/1-1.png" alt="1-1" style="zoom:50%;" />

---

本文主要讲解 Android Studio 集成开发环境 是怎么把项目的代码编译，链接，然后打包成 apk。这些知识相当重要，在排查各种编译问题的时候非常有用。

本文使用的 Android Studio 版本是 2021.3.1 

在 Windows 系统，vs2019 编译 C/C++ 项目使用的默认编译器是 `cl.exe`，链接器 `link.exe`。

在 Ubuntu 系统，clion 编译 C/C++ 项目使用的默认编译器跟链接器都是 gcc 命令。

那在 Windows 系统上，Android Studio 编译代码使用的是哪个命令呢？

---

本文先提供一个可以在 Android Studio 里面正常编译运行的项目 Hello，采用的语言是 Kotlin，点击 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/android/hello) 链接下载项目代码，然后放到 D 盘下面，如下：

![1-2](android_build\1-2.png)

然后直接把项目根目录下的 `build.gradle` 文件直接拖到 Android Studio 的界面上即可打开这个 Hello 项目，如下：

![1-3](android_build\1-3.png)

点击 Android Studio 菜单栏的绿色三角形即可把 Hello 这个 app 安装到 手机上，请先插好手机的数据线到电脑上。

运行效果如下：

![1-4](android_build\1-4.png)

<img src="https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/android_build/1-5.jpg" alt="1-5" style="zoom:25%;" />

这个 app 点击 按钮就会弹出一句提示 Hello FFmpeg Principle。

---

下面就来分析一下  Android Studio 编译，构建 Hello 这个 app 的过程。

在我个人的开发观念里面，我会把软件开发分为两种，命令行软件 和 GUI 界面软件。

1，命令行软件，这类软件是没有 漂亮的界面的。微软早期的 DOS 系统，LInux 系统上的命令行，Windows10 上面的 CMD 上跑的软件。例如 `ffmpeg.exe` 就是一个命令行软件。

这些软件可能是由 C/C++，Java 等等语言写出来的。这类软件的界面很简单，只需要有一个窗口显示文本信息即可，或者没有窗口。直接在后台运行也行，Windows 系统的后台服务是 Service，Linux 系统的后台服务器 Daemon（守护进程）。

命令行软件我觉得是最贴近**计算**本质的，因为它只使用了显卡/显示器的一部分能力，没有酷炫的动画，只有窗口文本。

做后台开发的程序员写代码，就是在用程序语言进行**计算**。假如把显卡，显示器从计算机里面拔除，剩下的功能就是**计算**了。例如把二进制数据从内存加载到寄存器，对寄存器的数据进行计算，然后再存储回去内存。

我觉得计算机里面主要有两个功能，**对数据进行计算** 与 **传输数据**，把数据从内存传到寄存器，把内存存储到硬盘，把内存数据传输给网络，传输给显示器，这些都属于传输数据。

---

在命令行软件开发的时候，相对来说是比较简单的，在各个系统平台上的移植性也比较好，因为 C/C++ 已经有很多标准库，各个平台都支持，编译器也可以把 C/C++ 翻译成各个CPU的指令集。

但是 GUI 界面软件的出现，把软件开发变得稍微复杂了。在早期 操作系统会把**显示这一部分功能** 封装成一些 API 函数给程序员使用，各个平台的 API 函数都不太一样，你要调很多 API 函数完成图片加载，布局，动画等功能。

例如 Windows 的 win32 界面API， 苹果公司的 Cocoa，Linux 的 GTK。

但是后面发现仅仅用 API 函数进行界面开发编程很繁琐，而且容易出错，例如忘记释放指针。所以逐渐演变成使用一些简单的语言来开发界面程序，例如 XML，HTML，XAML，等等。Android 里面用的就是 XML 来布局。

但是不要误解，CPU 不能直接执行 XML，XML 最终还是要翻译成函数调用，或者由虚拟层完成函数调用。从指令集角度看，函数调用就是一个 call。

因此可以说，是编译技术的发展 简化了 GUI 界面编程，因为编译技术的强大，才能把 XML 这么简洁的语法翻译成函数调用。

---

GUI 编程还有一个重点是，这种 app 或者 桌面软件 会有很多图片之类的应用资源，这些应用资源要通过另一个编译器（工具）来打包。例如 Android 的 AAPT2，Qt 的 rcc，C# 的 Resgen.exe，等等。

只要你做 GUI 编程，都会用到类似的工具来对应用资源进行编译打包。

应用资源编译器做的事情，相对于 C/C++，Java 的编译器来说，是比较简单的。应用资源的编译器主要 负责解析资源，为资源**编制索引**，让函数调用能更方便拿到图像数据，或者其他的应用资源。

---

Android Studio 编译应用资源和源代码，然后将它们打包成 APK 的构建过程如下：

<img src="https://ffmpeg.xianwaizhiyin.net/lib-ffmpeg/android_build/1-6.png" alt="1-6" style="zoom: 50%;" />



上图中的重点是，会把 class 文件翻译成  dex 文件，dex 文件里面的内容并不是原生的CPU 指令，还是虚拟机的字节码，只是 dex 的文件结构可以更加节省空间。

JAR Libraries 里面就是多个 class 文件的集合。

然后在 apk 安装到手机的过程中，会用 dex2oat 把 虚拟机字节码 翻译成原生的CPU指令。

具体的细节推荐阅读《[Dalvik 和 ART 有什么区别](https://juejin.cn/post/6844903748058218509)》《[10分钟了解Android项目构建流程](https://juejin.cn/post/6844903555795517453)》

---

Android APP 的编译构建过程是非常复杂繁琐的，需要执行很多步骤，每个步骤涉及到各种工具，编译器。

所以 Android Studio 默认会采用 [Gradle](http://gradle.org/) 来执行整个 APP 的构建，可以使用 Groovy 语言来编写 build 配置文件，让整个 APP 的构建变得更加简单。

可以把 Gradle 看成是 跟 CMake，Makefile 类似的构建系统。

关于 Gradle 的具体用法，推荐阅读《Gradle实战》一书。

---

因为构建系统已经把很多的细节封装起来，所以如果碰到一些编译构建问题。我们一定要学会看 Log，因为 Log 里面有最原始的命令，能看到更详细的报错。

我有时候解决编译问题，会去看 CMake 生成的 Makefile，查看 Makefile 里面的命令选项是否正常，因为这就是最原始的东西。利用这些信息反推出 CMake 的代码哪里写错了。

以 vs2019 为例，可以查看 Command Line 选项来看的最后执行的命令，如下：

![1-7](android_build\1-7.png)

或者点击 菜单栏的 Tool ➔ Options，把构建的日志级别设置成最大，重新编译项目即可看到 Log 里面最原始的命令。如下：

![msvc-static-1-13](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc-static/msvc-static-1-13.png)

```
C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Tools\MSVC\14.24.28314\bin\HostX86\x64\link.exe /ERRORREPORT:PROMPT /OUT:"C:\Users\loken\source\repos\c-single\x64\Debug\c-single.exe" /INCREMENTAL /NOLOGO kernel32.lib user32.lib gdi32.lib winspool.lib comdlg32.lib advapi32.lib shell32.lib ole32.lib oleaut32.lib uuid.lib odbc32.lib odbccp32.lib /MANIFEST /MANIFESTUAC:"level='asInvoker' uiAccess='false'" /manifest:embed /DEBUG:FASTLINK /PDB:"C:\Users\loken\source\repos\c-single\x64\Debug\c-single.pdb" /SUBSYSTEM:CONSOLE /TLBID:1 /DYNAMICBASE /NXCOMPAT /IMPLIB:"C:\Users\loken\source\repos\c-single\x64\Debug\c-single.lib" /MACHINE:X64 x64\Debug\hello.obj
```

---

那 Android Studio 类似的功能在哪里呢？怎么查看他构建过程中的 Log 呢？

答：开启 Gradle 的调试模式，这样 Gradle 就可以输出更信息的日志。点击 **File ➔ Settings **，在 Command-line Options 选项里输入 `--debug`，如下：

![1-8](android_build\1-8.png)

上面这样虽然设置了调试模式，但是我在 Log 日志里面还没没看到每个步骤使用的具体命令以及具体的选项，如下：

![1-9](android_build\1-9.png)

至于怎么才能看到具体命令以及具体的选项，后面补充。整个构建过程使用的命令 推荐阅读官方文档《[命令行工具](https://developer.android.com/studio/command-line?hl=zh-cn)》

---

由于 Android 项目里面会有 Java 代码，所以也需要 JDK 来编译 java 代码，那在哪里设置 java 编译器呢？如下：

![2-1](android_build\2-1.png)

我们可以进入 `C:\Program Files\Android\Android Studio\jre\bin` 目录一看，确实是有 `javac.exe` 编译器存在的。

![2-2](android_build\2-2.png)

---

现在已经大致了解了 Android APP 的构建过程。我们尝试不用 Android Studio，而是在命令行里面编译出来 apk 文件。

可以先将之前的 apk 文件删除，然后进入项目的根目录，如下：

```
D:
cd D:\FFmpeg-Principle\android\hello 
```

执行 assembleDebug 任务，如下：

```
.\gradlew.bat assembleDebug
```

![2-3](android_build\2-3.png)

任务执行完毕之后，可以搜索 `*.apk` ，就会发现多了一个 apk 文件出来。

![2-4](android_build\2-4.png)

官网文章《[从命令行构建您的应用](https://developer.android.com/studio/build/building-cmdline?hl=zh-cn)》详细讲解了如何从命令行里面构建项目。

---

本文对 Android 项目的构建过程已经介绍完毕，关于 Android Studio 开发环境的使用，强烈推荐阅读官网的文档《[Android Studio用户指南]()》或者《精通Android Studio》一书。

而 Gradle 跟 Groovy 可以买本 《Gradle 实战》来看看

---

参考资料：

1，《[10分钟了解Android项目构建流程](https://juejin.cn/post/6844903555795517453)》

2，《[Android构建流程](https://blog.csdn.net/dbs1215/article/details/106888164)》

3，《[Dalvik 和 ART 有什么区别](https://juejin.cn/post/6844903748058218509)》



