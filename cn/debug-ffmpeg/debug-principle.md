# 调试基础知识及原理—FFmpeg调试环境搭建

<div id="meta-description---">一个可调试的可执行文件。我个人觉得里面的内容可以分为3个部分。
1，机器码。C/C++ 转成的机器码
2，符号表信息（symbols）
3，调试信息（debug info）
无论是 Linux 的GDB，还是 Windows 的 WinDbg 跟 VsDebug，都是根据上面这些信息来进行调试的。</div>

一个可调试的可执行文件。我个人觉得里面的内容可以分为3个部分。

1，机器码。C/C++ 转成的机器码

2，符号表信息（symbols）

3，调试信息（debug info）

无论是 Linux 的GDB，还是 Windows 的 WinDbg 跟 VsDebug，都是根据上面这些信息来进行调试的。



------

用前面 [ubuntu18 + clion](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/ubuntu18-clion.html) 编译出来的 ffmpeg_g 文件来讲解一下调试器的工作过程。ffmpeg_g 是一个 ELF 文件，用 [xelfviewer](https://github.com/horsicq/XELFViewer) 打开，如下：

![debug-principle-1-1](debug-principle\debug-principle-1-1.png)

上图是是 ffmpeg_g 里面的符号表，`init_options`  是里面的一个符号，一个函数就是一个符号。

上图中圈出来的 `init_options` 是 `ffmpeg_opt.c` 里面的一个函数。我编译 ffmpeg_g 的时候 开启了最大的调试信息，所以 symbols 跟 debug info 都有。

下面用 gdb 来调试一下 这个 ffmpeg_g 程序，如下：

```
gdb ./ffmpeg_g
set args -i walking-dead.mp4 walking-dead.flv -y
b init_options
layout src
r
```

![debug-principle-1-2](debug-principle\debug-principle-1-2.png)

上图里面，`b init_options` 就是针对 `init_options` 这个符号打断点 ，elf 里面必须有符号表才能根据 符号打断点，如果把符号表删了就无法指定符号断点。

然后，这个符号对应的源代码行数，是在 ELF 里面的 debug_xxx 的段表里面的，这是按照 DWAF （Debug With Arbitrary Record Format）标准存储的，如下：

![debug-principle-1-3](debug-principle\debug-principle-1-3.png)

可以使用 `nl -l ffmpeg_g` 打印符号对应的源代码行数。

**ELF 文件生成的时候，符号 跟 源代码的对应关系已经写死了。**

例如  `init_options` 符号就是在 `/home/ubuntu/Documents/FFmpeg-n4.4.1/fftools/ffmpeg_opt.c:223` 的位置，如果你把源文件目录改了位置，GDB 查询 ELF 的 debug_xxx 段表的时候就会找不到源文件。

------

因此，文章刚开始提及的 符号表信息（symbols）就是变量名，函数名之类的，调试信息（debug info）是 函数对应的源代码信息。

现在做个测试，我们把 debug info 信息删掉，只保留 symbols ，GDB 调试的时候会有什么特别。

执行以下命令 删除 调试信息：

```
strip --strip-debug ffmpeg_g
```

![debug-principle-1-4](debug-principle\debug-principle-1-4.png)

从上图可以看到，删除调试信息之后，文件小了44M。再用 xelfviewer 打开 删除了 debug info 的 ffmpeg_g 文件，可以发现 之前的 debug_info，debug_line 这些段表全都不见了，如下：

![debug-principle-1-4-1](debug-principle\debug-principle-1-4-1.png)

现在再次执行以下 gdb 命令调试。

```
gdb ./ffmpeg_g
set args -i walking-dead.mp4 walking-dead.flv -y
b init_options
layout src
r
```

![debug-principle-1-5](debug-principle\debug-principle-1-5.png)

从上图可以看到 `b init_options` 还是可以断点成功，因为我们只是删了调试信息，符号信号还存在，只是 gdb 的提示有变化，只是提示这个函数的地址，之前有调试信息的时候，gdb 会直接提示 ffmpeg_opt.c:223。

这是因为我们把调试信息删了，gdb 找不到这个 init_options 符号对应的源代码信息。

因此，`layout src` 也会失败，提示没有 源信息，如下：

![debug-principle-1-6](debug-principle\debug-principle-1-6.png)

因此，只能使用汇编调试，执行 `layout asm` ，切换到汇编界面。

有符号信息的汇编调试，跟无符号信息的汇编调试有什么区别？请看下图：

![debug-principle-1-7](debug-principle\debug-principle-1-7.png)

有调试符号的好处就是，call 那些地址，gdb 都会显示一个函数名给你看，然后 执行 bt 栈回溯的时候，也能显示出来函数名称。

虽然看不到源码，但是能看到函数名，微软经常提供这种符号信息给别人调试，但是不开放debug info 信息跟源码。

------

现在我再把 符号表也删掉，命令如下：

```
strip --strip-all ffmpeg_g
```

![debug-principle-1-8](debug-principle\debug-principle-1-8.png)

从上图可以看出，符号表信息相对较少，只有1M。现在再用 gdb 调试一下 ffmpeg_g，命令如下：

```
gdb ./ffmpeg_g
set args -i walking-dead.mp4 walking-dead.flv -y
```

![debug-principle-1-9](debug-principle\debug-principle-1-9.png)

从上图可以看出，`init_options` 符号已经删掉了，无法根据这个符号进行断点。只保留了一个 main 让你能从入口断一下。使用 bt 回溯的时候，也只能看到一串地址，没有函数名，这时候的 ffmpeg_g 只能看汇编调试，犹如天书。

------

讲到这里，ffmpeg_g 软件的运行 其实只需要 机器码，符号表信息（symbols）跟 调试信息（debug info）都只是调试需要。

因此我个人把 symbols 看成是初级的调试信息， debug info 是高级调试信息。

因为  debug info 要结合源码来用，所以即使你有debug info，如果没有源码，用处不是很大，但是对于逆向分析，有 debug info 总比没有好。

本文讲的是 Linux 的ELF 调试情况，Windows 的PE 也是类似的。 PE 跟 ELF 事实上同根同源，他们都是由 COFF （Common Object File Format） 格式发展来的。两者都是基于 段的结构。

ELF 相关知识可以看博客文章 《[ELF格式简介](https://www.xianwaizhiyin.net/?p=2179)》

------

参考资料：

1，《软件调试》- 张银奎

2，《Linker and Loader》-  John R. Levine

3，《程序员的自我修养 - 链接、装载与库》- 俞甲子，石凡，潘爱民
