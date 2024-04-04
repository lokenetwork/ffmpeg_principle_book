# MSVC编译动态库—编译链接基础知识

<div id="meta-description---">MSVC编译动态库</div>

本文跟[《Linux环境编译动态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-shared.html) 使用同样的项目 universe，下载之后，放到 D盘，如下：

![msvc-multiple-1-1](msvc-multiple\msvc-multiple-1-1.png)

编译链接命令如下：

```
cl.exe /c earth.c moon.c sun.c 
link.exe /DLL /DEBUG /EXPORT:sun_rotate /EXPORT:moon_rotate /EXPORT:earth_rotate /OUT:star.dll earth.obj moon.obj sun.obj 
cl.exe /c zeus.c
link.exe /DEBUG /OUT:zeus.exe zeus.obj star.lib 
```

![msvc-shared-1-1](msvc-shared\msvc-shared-1-1.png)

我们再用 dumpbin 来查看一下 zeus 是否依赖 star.dll ，如下：

```
dumpbin /DEPENDENTS zeus.exe
```

![msvc-shared-1-2](msvc-shared\msvc-shared-1-2.png)



------

Windows 的动态库跟 Linux 的动态库有点不一样，Windows 使用动态库需要一个 lib 导入库来支持编译。如下：

![msvc-shared-1-3](msvc-shared\msvc-shared-1-3.png)

上图中，这个 `star.lib` 不是静态库，而是动态库的导入库（import library）。是给 `link.exe` 链接器使用的，链接器用 `star.lib` 来解析源代码中的函数调用。

`star.lib` 不包含任何代码，它只是给链接器提供信息，这些信息可以用来建立 exe 文件里面的重定位表，重定位表（relocation tables）是给动态链接进行使用的。

只有链接的时候才需要 `star.lib` ，生成 exe 之后可以把 `star.lib` 删掉，也不会影响 exe 的运行。exe 的运行只需要 `star.dll` 。

`star.lib` 是通过 上面命令的 `/EXPORT:xxx` 导出来的，上面的命令中，我导出了 3个函数，所以要写 3次，很麻烦，其实 `link.exe` 也支持 [DEF](https://docs.microsoft.com/zh-cn/cpp/build/reference/def-specify-module-definition-file?view=msvc-170) 文件的方式导入。

在 universe 目录新建一个 star.def 文件，内容如下：

```
EXPORTS
    sun_rotate
    moon_rotate
    earth_rotate
```

然后执行以下命令重新编译动态库：

```
link.exe /DLL /DEBUG /DEF:star.def /OUT:star.dll earth.obj moon.obj sun.obj 
```

上图中的 star.pdb 实际上是调试文件，windows 是把调试信息单独文件存放的。只有使用了 /DEBUG 选项，才会生成 pdb 文件。

提前剧透一下，msys2 + msvc 编译 FFmpeg 的时候，FFmpeg 的动态库函数也是通过 def 文件导出来的，如下：

![msvc-shared-1-4](msvc-shared\msvc-shared-1-4.png)

![msvc-shared-1-5](msvc-shared\msvc-shared-1-5.png)

------

以上都是使用 cl.exe 跟 link.exe 手动生成 动态库，现在用 vs2019 演示一下如何创建跟编译一个动态库。

由于 vs2019 的默认动态库项目会引入一些其他的配置，我们只是简单的 3 个C文件，所以不选 DLL 项目，直接创建 空白项目，然后改项目类型就行。

![msvc-shared-1-6](msvc-shared\msvc-shared-1-6.png)

创建一个空白项目，把项目命名为 **star** 。

![msvc-static-1-5](msvc-static\msvc-static-1-5.png)

然后再打开项目配置，设置 Platform 跟 General ➜ Configuration Type，如下：

![msvc-shared-1-7](msvc-shared\msvc-shared-1-7.png)

然后把 3 个文件 earth.c moon.c sun.c 复制到 star 目录，如下：

![msvc-shared-1-8](msvc-shared\msvc-shared-1-8.png)

然后点击 Add ➜ Exiting Item 添加3个文件进去解决方案，如下：

![msvc-shared-1-9](msvc-shared\msvc-shared-1-9.png)

这样默认只会生成 star.dll ，但是不会生成 star.lib 导入库的，因此我们把 star.def 文件也拷贝过去项目目录（跟C文件同目录），

然后设置 Linker ➜ All Options ➜ Module Definition File  ，如下：

![msvc-shared-1-10](msvc-shared\msvc-shared-1-10.png)

应用之后，就会发现 Command Line 里面就出现 `/DEF:"star.def"` 选项了。

![msvc-shared-1-11](msvc-shared\msvc-shared-1-11.png)

**重点： Command Line 特别好用，无论是 C/C++ 的编译参数，还是 lib.exe 生成静态库的参数，还是 link.exe 编译exe或者动态库的参数。都在 各自的 Command Line ，你可以通过这个  Command Line 查看实际运行的参数，看看有没问题，不需要rebuild 从日志里面看。**

------

现在再重新 rebuild 一下，就会生成 star.lib 跟 star.dll 了，如下：

![msvc-shared-1-12](msvc-shared\msvc-shared-1-12.png)

参考资料：

1，[《MSVC EXPORT用法》](https://docs.microsoft.com/zh-cn/cpp/build/reference/export-exports-a-function?view=msvc-170)
