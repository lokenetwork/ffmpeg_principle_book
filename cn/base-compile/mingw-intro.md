# MinGW介绍—编译链接基础知识

<div id="meta-description---">MinGW介绍</div>

MinGW 主要是是把 GCC 编译器以及一些 配套的命令 objdump 跟 readelf 也移植到 Windows 平台。官网地址：www.mingw-w64.org

参考 C语言中文的的教程 [《MinGW下载和安装教程》](http://c.biancheng.net/view/8077.html)安装一下 。 安装在 C盘，如下：

![mingw-intro-1-1](mingw-intro\mingw-intro-1-1.png)

我们查看一下这个 MinGW 目录下面有什么，如下：

![mingw-intro-1-2](mingw-intro\mingw-intro-1-2.png)

可以看到，有很多 跟 编译器相关的 软件。这些软件可以在 WinCMD 下直接运行的，我们进入 命令行，如下：

![mingw-intro-1-3](mingw-intro\mingw-intro-1-3.png)



------

还可以发现，在 lib 目录下，有很多的 .a 跟 .o 文件，如下：

![mingw-intro-1-4](mingw-intro\mingw-intro-1-4.png)

.a 文件不用管，实际上目前 Linux 的 .a 文件跟 Windows 的 .lib 文件，格式是完全一样的，都是打包。

我们想知道这些 .o 文件 跟 msvc 编译出来的 .obj 文件是不是一样的，用 dumpbin 开看一下，可以解析出信息，文件格式是 COFF ，所以 MinGW 里面的 .o 实际上就是 .obj 文件，只是变了一下后缀，因为 gcc 编译出来的 就是 .o 后缀。

![mingw-intro-1-5](mingw-intro\mingw-intro-1-5.png)

提示，在 Linux 下用 gcc 编译出来的 .o ，是无法用 dumpbin 解析的，因为 Linux 的 .o 是 ELF 格式。

------

在 MinGW 目录下创建一个 项目 projects/c-single ，还是以 hello.c 演示一下 gcc 的用法。

![mingw-intro-1-6](mingw-intro\mingw-intro-1-6.png)

使用以下命令编译 hello.c 

```
cd C:\MinGW\bin
.\gcc.exe -c -o C:\MinGW\projects\c-single\hello.o C:\MinGW\projects\c-single\hello.c
```

初学者我不建议你 配置 MinGW的bin目录为环境变量，因为会影响文章后面的 MSYS2 的环境。

然后再链接 hello.o 为 exe 文件，如下：

```
.\gcc.exe -o C:\MinGW\projects\c-single\hello.exe C:\MinGW\projects\c-single\hello.o
```

运行结果如下：

![mingw-intro-1-7](mingw-intro\mingw-intro-1-7.png)

再用 dumpbin 查看一下这个 hello.exe 的依赖，如下：

```
dumpbin /DEPENDENTS C:\MinGW\projects\c-single\hello.exe
```

![mingw-intro-1-8](mingw-intro\mingw-intro-1-8.png)

从上图可以看出，MinGW 的 gcc 编译出来的 exe 非常 Windows 原生，跟 cl.exe 简直一模一样。

------

之前提到 hello.o 实际上跟 hello.obj 是类似的，那可不可以直接用 link.exe 来把 hello.o 链接成 exe 呢？试一下。

```
cd C:\MinGW\projects\c-single\
link.exe /DEBUG /OUT:hello.exe hello.o
```

![mingw-intro-1-9](mingw-intro\mingw-intro-1-9.png)

可以看到，报错了。

因此 MinGW 跟 MSVC 的编译器 ，链接器 不要交叉使用。

虽然 .o 跟 .obj 文件格式一样，但是内部的内容，还是有点区别的。因为他们的链接器需要不同的内容。
