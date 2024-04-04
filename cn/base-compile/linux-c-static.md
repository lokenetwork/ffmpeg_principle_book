# Linux环境编译静态库—编译链接基础知识

<div id="meta-description---">Linux环境编译静态库</div>

还是以之前的 universe 项目为例，之前是 `zeus.c` （宙斯）可以操作 3 颗星球。现在来了一个新的大佬 `poseidon.c` （波塞冬），他也可以操作 3 颗星球。

为了方便，我们需要把 `moon.o`、`sun.o` 、`earth.o` 这 3 个东西弄成一个东西。这种方法 就是 静态库 跟 动态库，静态库 是可以链接进去 程序自身，动态库是共享库，可以由多个程序共享节省空间，动态库只有用到的时候才会加载。

调用静态库里面的函数，通常比调用动态库里面的函数会更高效一些。因为从汇编的角度看，调用静态库的函数只有一条 `call` 指令就直接跳到函数的地址了，而动态库的方式，需要两次 `call`，第一次是从 **Procedure Linkage Table 表** 先找到函数的地址。

本文只讲静态库封装，动态库下一篇文章再讲。

------

生成静态库的命令如下：

```
ar -rcs libstar.a moon.o sun.o earth.o
```

上面的命令把 3 个 `.o` 封装打包进去 `libstar.a` 里面了，`star` 是星球的意思，这是星球静态库。

`ar` 实际上是一个打包封装命令，打包成静态库，静态库也叫归档文件，实际上是对已经编译好的字节码进行封装，封装过程不涉及到链接，以前那些 `call` 占位符还是 00 00 。

`ar` 不止可以打包 `elf` 的 `.o` 文件，还可以打包文本文件，如下命令：

```
ar -r text-all.arch a.txt b.txt
```

![linux-c-static-1-1](linux-c-static\linux-c-static-1-1.png)

ar 甚至支持 Windows 的PE 格式，使用 `ar --help` 可以查看 ar 支持的所有打包格式。

> ```
> ar: supported targets: elf64-x86-64 elf32-i386 elf32-iamcu elf32-x86-64 a.out-i386-linux pei-i386 pei-x86-64 elf64-l1om elf64-k1om elf64-little elf64-big elf32-little elf32-big pe-x86-64 pe-bigobj-x86-64 pe-i386 plugin srec symbolsrec verilog tekhex binary ihex
> ```

------

回到 文本主题，静态库 libstar.a 是什么格式的呢？ELF格式吗？实际上，他不是一个纯正的 ELF 格式，因为用 xelfviewer 无法解析，但是 readelf 命令可以查看内容。这是因为 ar 命令把多个ELF文件合并在一起了。

我们用 objdump 查看 libstar.a 里面的字节码对应的汇编指令，如下：

```
objdump -d libstar.a > star-dump.txt
```

![linux-c-static-1-2](linux-c-static\linux-c-static-1-2.png)

sun.o 里面是 调了 moon.o 里面的 `moon_rotate()` 函数的，即便我们把 moon.o 跟 sun.o 一起打包成静态库，这个地址还是没有被修正，**所以编译成静态库并不会进行链接操作**。

------

现在 libstar.a 静态库已经生成了，如何引用这个静态库呢？之前说过，创建静态库 是为了给  zeus.c （宙斯） 跟 poseidon.c （波塞冬）使用。

先在 Document 里面创建一个 poseidon 文件夹，再创建一个 poseidon.c 文件，poseidon.c 内容如下：

```
#include "sun.h"
#include "moon.h"
#include "earth.h"
#include <stdio.h>
int main()
{
    printf("poseidon do something \r\n");
    sun_rotate();
    moon_rotate();
    earth_rotate();
    return 0;
}
```

把 相关的头文件 跟  libstar.a 复制过去，目录结构如下：

![linux-c-static-1-3](linux-c-static\linux-c-static-1-3.png)

编译 poseidon.c  的命令如下：

```
gcc -c -o poseidon.o poseidon.c 
```

第一种引用 libstar 静态库的 方式是 直接写 静态库文件的全称，会在当前目录找 libstar.a， 命令如下：

```
gcc -o poseidon poseidon.o libstar.a
```

第二种方式是是  ` -static` 指定后面的库是静态库，然后使用 `-l` 参数 指定库名称，命令如下：

```
gcc -v -o poseidon poseidon.o -static -lstar
```

上面的命令，如果不指定 `-static` ，链接的时候会默认去搜索动态库，也就是搜索 libstar.so 文件。

使用 `-l` 链接的时候，需要去除库文件名的 lib 前缀 跟 .a 后缀，去掉就行，链接器会默认加上这些前缀后缀找到那个库文件的。运行结果如下：

![linux-c-static-1-4](linux-c-static\linux-c-static-1-4.png)

上图使用了 -v 选项，所以打印了更详细的内容，包括 gcc 会在哪些目录搜索 库文件。

从上图可以看出，gcc 会在 上面 `/usr/lib` 等等 那些目录里面查找 libstar.a 文件。上图的命令是会报错的，因为那些目录都没有 libstar.a 。

我现在把 libstar.a 移动到 `/usr/lib` 目录，链接就能通过了，请看下图：

![linux-c-static-1-5](linux-c-static\linux-c-static-1-5.png)

编译通过之后，即使把 libstar.a 文件删除，poseidon 程序运行也不会出问题，因为字节码已经拷贝过去了。

还有一种链接静态库的方式，直接把路径写全。

```
gcc -o poseidon poseidon.o /usr/lib/libstar.a
```

------

poseidon 项目已经编译完毕了，现在再在 Document 建立一个 zeus 文件夹，把文件复制过去，如下：

![linux-c-static-1-6](linux-c-static\linux-c-static-1-6.png)

用以下命令开始编译。

```
gcc -c -o zeus.o zeus.c
gcc -o zeus zeus.o /usr/lib/libstar.a
```

![linux-c-static-1-7](linux-c-static\linux-c-static-1-7.png)

这样，两位大佬 zeus 跟 poseidon 都能使用这个 libstar.a 静态库了。

**总结：Linux 的静态库封装，实际上就是打包，把多个 .o 文件打包在一个 .ar 文件里面。这也是为什么静态库会被叫做归档文件的原因**

------

扩展知识：

实际上，编译 跟 链接，是可以合成一步的，上面 zeus 的 两条命令，可以合成一条。如下：

```
gcc -o zeus zeus.c /usr/lib/libstar.a
```

这样不会生成 zeus.o 文件，但是实际上在内部，还是有一个编译步骤的。我不建议初学者这样用，因为这个命令很容易混淆 编译 跟 链接这两个过程。

------

上面已经讲解了，找不到静态库，gcc 会报如下错误。

![linux-c-static-1-8](linux-c-static\linux-c-static-1-8.png)

但是编译大型 C/C++ 项目的时候，还会遇到 undefined reference to `xxx'，如下：

![linux-c-static-1-9](linux-c-static\linux-c-static-1-9.png)

这个错误的意思是，函数被引用了，但是具体的实现没有找到。为什么会报这个错误呢？是因为我编译的时候 故意不写上 `-static -lstar`。

在实际项目中，通常是写漏了库引用，或者名称没写对了，例如写成 -lstarx ，多了个x，但是 libstarx.a 可能是存在的，所以就没有报找不到库，而是 undefined reference to `xxx' 。

