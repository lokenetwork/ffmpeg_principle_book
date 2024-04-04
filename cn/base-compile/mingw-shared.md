# MinGW编译动态库—编译链接基础知识

<div id="meta-description---">MinGW编译动态库</div>

以之前的  [universe](https://pan.baidu.com/s/1_reA14rpTZ6PTwY7Qc_q2Q) 项目为例，提取码：mku9 。请下载后放到 `C:\MinGW\projects` 目录，如下：

![mingw-static-1-1](mingw-static\mingw-static-1-1.png)

现在用 MinGW 的 gcc 来编译出 libstar.so 动态库给 zeus 使用，如下：

```
cd C:\MinGW\bin
.\gcc.exe -c -o C:\MinGW\projects\universe\earth.o C:\MinGW\projects\universe\earth.c
.\gcc.exe -c -o C:\MinGW\projects\universe\sun.o C:\MinGW\projects\universe\sun.c
.\gcc.exe -c -o C:\MinGW\projects\universe\moon.o C:\MinGW\projects\universe\moon.c
.\gcc.exe -fPIC -shared -o C:\MinGW\projects\universe\libstar.so C:\MinGW\projects\universe\sun.o C:\MinGW\projects\universe\moon.o C:\MinGW\projects\universe\earth.o
```

上面这些命令，参数跟 在 [《Linux环境编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-shared.html)是一样的。都是用  `-shared`  生成动态库。

![mingw-shared-1-1](mingw-shared\mingw-shared-1-1.png)

然后执行以下命令来使用这个 libstar.so 动态库，如下：

```
.\gcc.exe -c -o C:\MinGW\projects\universe\zeus.o C:\MinGW\projects\universe\zeus.c
.\gcc.exe -o C:\MinGW\projects\universe\zeus2.exe C:\MinGW\projects\universe\zeus.o C:\MinGW\projects\universe\libstar.so
```

![mingw-shared-1-3](mingw-shared\mingw-shared-1-3.png)

上图中，程序运行成功了，这个 exe 的依赖如下：

![mingw-shared-1-4](mingw-shared\mingw-shared-1-4.png)



------

现在，我们有个以疑问，这个 libstar.so 到底是个什么东西？他的后缀名是 .so ，那他的文件格式就是 Linux 的 ELF 吗？

用 PEview 来查看这个 libstar.so 即可知道，如下：

![mingw-shared-1-2](mingw-shared\mingw-shared-1-2.png)

上图中，这个文件开头是 MZ，所以他是一个 PE 格式的文件，.so 后缀只是挂羊头卖狗肉。

在 前文 [《MSVC编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc-shared.html) 编译Windows 原生动态库的时候，是有两个文件的，一个 lib （导入库），一个 dll （动态库）。

而 MinGW 生成的只有一个 libstar.so 文件，而 MinGW 也是偏 Windows 原生，那肯定是 lib 跟 dll 的东西都放在 libstar.so 里面了。

从 上图中的 dumpbin 依赖中也可以看出，libstar.so 实际上也是一个 dll。

------

 libstar.so 这种集成的格式只是适合 gcc 链接器使用，而 MSVC 使用动态库需要 lib 跟 dll。有没一种办法，可以让 MinGW 编译出 lib（导入库） 跟 dll （动态库）呢？

```
.\gcc.exe -fPIC -shared -o C:\MinGW\projects\universe\star.dll C:\MinGW\projects\universe\sun.o C:\MinGW\projects\universe\moon.o C:\MinGW\projects\universe\earth.o -Wl,--out-implib,C:\MinGW\projects\universe\star.lib
```

上面我把 so 后缀改 dll 了，但是实际上没有区别。主要是后面的 `-Wl,--out-implib` 选项，这里生成了 lib 导入库。

上面的命令我没有指定 def 需要导出哪些函数，默认是全部都导出到 star.lib 。

现在的目录文件如下：

![mingw-shared-1-5](mingw-shared\mingw-shared-1-5.png)

------

这样，MSVC 的 cl.exe 跟 link.exe 就能使用 gcc 生成的 star.lib 跟 star.dll 了。

由于上面 gcc 编译的默认是 32 位的，所以使用 **x86 vs2019** 的环境，不要用 64 的。

用 x86 Native Tools Command Prompt for VS 2019 打开一个新窗口，然后 进入 universe 目录，执行以下命令 ：

```
cd C:\MinGW\projects\universe
cl.exe /c zeus.c
link.exe /OUT:zeus3.exe zeus.obj star.lib
```

![mingw-shared-1-6](mingw-shared\mingw-shared-1-6.png)

正常运行，所以 gcc 编译出的动态库 ，是可以给 msvc 使用的。这是因为 这两个 编译器 处理 C语言程序 是 ABI 兼容的。但是 C++ 程序不是，gcc 跟 cl.exe 编译器编译 C++ 动态库，经常会遇到 ABI 兼容问题，具体下篇文章再讲。
