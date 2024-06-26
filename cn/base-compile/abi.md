# 什么是ABI二进制兼容—编译链接基础知识

<div id="meta-description---">ABI兼容这个问题，主要是跟 编译器有关，跟链接器是没关系的，链接器只负责把字节码链接起来。只要 不同编译器生成的字节码是 API 兼容的。链接器就可以把他们链接起来，不过对于不同的格式，链接器还有很多工作要做。</div>

ABI兼容这个问题，主要是跟 编译器有关，跟链接器是没关系的，链接器只负责把字节码链接起来。

只要 不同编译器生成的字节码是 API 兼容的。链接器就可以把他们链接起来，不过对于不同的格式，链接器还有很多工作要做。

比如之前的 [《MinGW介绍》](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-intro.html) 中，gcc 编译出来的 .o 文件，不能被 msvc 的 link.exe 使用，实际上他们的 字节码在 ABI 角度看是兼容的，只是格式没处理好。

但是 .o 文件也不能说是 导出了 二进制接口，导出动态库才是导出 ABI 接口给别人用。



------

ABI 的全称是 Application Binary Interface，也就是应用程序二进制接口。什么是二进制接口？

理解ABI兼容需要比较多的汇编知识积累，推荐阅读《汇编语言 基于x86》一书。

无论是 C/C++ ，最后都是翻译汇编指令 给 CPU 执行的。这里我简单讲一些 汇编 的基础知识，

汇编程序 能干的活，主要就是加减乘除，然后复制内存，拷贝内存，修改内存数据，操作寄存器，寄存器也是一种内存。内存是二进制的东西，一串01的比特流。

汇编里面有个 call 指令，调用某个函数，实际上就是跳转到某个地址。call 之前，你要告诉这个函数，一些必要的信息。

例如，有个 C 语言函数如下：

```
int do_job(int num){

}
```

上面的 do_job 函数 接受 num 参数，但是 call do_job 之前，这个 num 的数据放在哪里呢？放在 eax 寄存器，还是 ebx 寄存器，还是栈里面？

如果 编译器 gcc 把 这个 num 放在 eax 寄存器，然后调 do_job，但是 do_job 的代码是 cl.exe 生成的，cl.exe 是从栈里面拿参数，就乱套了。

提醒：传参是调用者做的事情。

因此必要要有标准，参数放在哪里？

上面只是打个比方，实际上 C语言的传参已经有标准。

参数的内存布局只是影响 ABI 兼容的一个因素，还有结构体对齐等等一些也会影响，需要统一。

------

字节码指令操作的都是内存数据，所以 编译器编译生成 C/C++ 程序的时候，**各种数据的内存空间布局统一了才是 ABI 兼容**。

而 C++ 有很多复杂的语法，不同编译器直接做到 ABI 兼容比较麻烦。不过 C++ 的跨平台性很好，可以做到 源码级 兼容。但是对于一些不愿意提供源码的服务商不太友好。

------

下面摘自《程序员的自我修养》第4.5章节。

> 所以人们一直期待着能有统一的C++二进制兼容标准（C++ ABI），诸多的团体和社区都在致力于C++ ABI标准的统一。但是目前情况还是不容乐观，基本形成以微软的VISUAL C++和GNU阵营的GCC（采用Intel Itanium C++ ABI标准）为首的两大派系，各持己见互不兼容。早先时候，*NIX系统下的ABI也十分混乱，这个情况一直延续到LSB（Linux Standard Base）和Intel的Itanium C++ ABI标准出来后才有所改善，但并未彻底解决ABI的问题，由于现实的因素，这个问题还会长期地存在，这也是为什么有这么多像我们这样的程序员能够存在的原因。

![abi-1-1](abi\abi-1-1.png)

> 如果能够保证上述4种情况不发生，那么绝大部分情况下，C语言的共享库将会保持ABI兼容。注意，仅仅是绝大部分情况，要破坏一个共享库的ABI十分容易，要保持ABI的兼容却十分困难。很多因素会导致ABI的不兼容，比如不同版本的编译器、操作系统和硬件平台等，使得ABI兼容尤为困难。使用不同版本的编译器或系统库可能会导致结构体的成员对齐方式不一致，从而导致了ABI的变化。这种ABI不兼容导致的问题可能非常微妙，表面上看可能无关紧要，但是一旦发生故障，相关的Bug非常难以定位，这也是共享库很大的一个问题。

