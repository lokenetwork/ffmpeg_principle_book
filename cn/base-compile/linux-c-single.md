# Linux环境编译单个C程序文件—编译链接基础知识

<div id="meta-description---">Linux环境编译单个C程序文件</div>

我的 Linux 环境是 Ubuntu18，Linux 环境用到的编译链接工具 就是 gcc 命令。

首先 在 Document 目录下 创建一个 c-single 目录 作为本文的项目目录，再创建一个 hello.c 文件

hello.c 代码如下：

```
#include <stdio.h>
int main()
{
    printf("hello ffmpeg \r\n");
    return 0;
}
```

截图如下：

![linux-c-single-1-1](linux-c-single\linux-c-single-1-1.png)



------

执行以下 命令开始编译。

```
gcc -c -o hello.o hello.c
```

讲解一下 gcc 命令的用法， `-c` 是指只编译程序，不进行链接， `-o` 是指定输出文件名，hello.c 就是输入文件。

gcc 的语法规则是这样的， `-o` 后面就是输出文件，如果一个 参数前面什么都没有，那他就会被当做输入，hello.c 前面就什么都没有。可以有多个输入文件，编译出一个输出文件。

![linux-c-single-1-2](linux-c-single\linux-c-single-1-2.png)

编译之后会生成 hello.o 目标文件，如下：

![linux-c-single-1-3](linux-c-single\linux-c-single-1-3.png)

从上图可以看出来，hello.o 文件是一个二进制文件，ELF格式。这里推荐用 xelfviewer 来查看，如下：

![linux-c-single-1-4](linux-c-single\linux-c-single-1-4.png)

------

hello.o 里面已经包含了 hello.c 对应的字节码，所以我们反编译看一下 hello.o 里面的字节码内容，执行以下命令。

```
objdump -d hello.o
```

![linux-c-single-1-5](linux-c-single\linux-c-single-1-5.png)

上图我圈出来了一个 call 指令，这里就是 调 printf 函数的指令。但是这个 地址是 00 00 00 ，这肯定是不对的，现在只是一个占位，占了 4个字节，所以这个函数的地址是 4字节大小。

后面 链接器 会通过上面的 .rela.txt （重定位段）来修正 这个 printf 符号的地址，因为这个符号的地址 是在 C 标准库 libc 里面的，链接器需要扫描 libc 库，才能知道这个 符号的具体地址。

虽然本文编译的是单个 C 文件，但是还是会链接其他的外部库的，这个外部库 就是 libc，也叫做**运行时库**，是 gcc 链接器默认给你加上的，不用指定。

运行时库 这个词 有点不太好理解，你可以简单把它看成是一个外部的第三方库即可，虽然 运行时库跟外部库有一点区别。

------

现在开始**链接程序** ，执行以下命令。

```
gcc -o hello hello.o
```

![linux-c-single-1-6](linux-c-single\linux-c-single-1-6.png)

再用 objdump 看一下，会发现，**之前的 call 的地址已经被修正 成了 c6 fe ff ff**，如下：

![linux-c-single-1-7](linux-c-single\linux-c-single-1-7.png)

------

我们知道，C/C++ 引入一个外部库的时候，这个外部库可以是动态库，也可以是静态库。

那 hello 引用 的 libc 是以静态还是动态的方式引入的呢？可以通过以下命令查看。

```
ldd hello
```

![linux-c-single-1-8](linux-c-single\linux-c-single-1-8.png)

从上图 可以看出来， libc 是以动态库的方式链接给 hello 的。**这是因为 gcc 默认是使用动态链接，如果找不到 动态 so 库，才会找 .a 静态库来链接。**

那可不可以**指定**使用静态的方式 把 libc 链接给 hello呢？也是可以的，命令如下：

```
gcc -o hello hello.o -static -lc
```

`-static` 这个选项 会导致所有库都以静态的方式链接，`-l` 代表这是一个需要连接的库，不需要加 "lib" 。具体请看[《LD链接器文档》](https://ftp.gnu.org/pub/old-gnu/Manuals/ld-2.9.1/html_node/ld_3.html#SEC3)

这时候，再用 `ldd`  查看 信息，会发现 已经没有了 libc.so 这个字符。

![linux-c-single-1-8-1](linux-c-single\linux-c-single-1-8-1.png)

这里讲一个扩展知识点，Windows 跟 Linux 静态链接 C运行时库的方式不太一样。**Windows 是在编译阶段 通过 /MT 或者 /MD 指定的**，如下：

```
cl.exe /D_ISOC99_SOURCE /MT /c /Fo./hello-mt.o hello.c
link.exe /OUT:./hello-mt.exe ./hello-mt.o
cl.exe /D_ISOC99_SOURCE /MD /c /Fo./hello-md.o hello.c
link.exe /OUT:./hello-md.exe ./hello-md.o
```

而 Linux 的 gcc 编译阶段不需要指定 库的链接方式，是在 链接阶段 才确定是静态库还是动态库。

------

现在对比一下 两种方式编译的 hello，如下：

![linux-c-single-1-9](linux-c-single\linux-c-single-1-9.png)

从上图可以看出， hello-static 比 hello-shared 大了 800kb，但是 libc.a 静态库明明有 5.3M，这是怎么回事呢？

这是因为 静态库 实际上是一个 归档（archive）文件，静态库里面可能包含了1000多个函数的实现代码，但是 hello 可能只用到 20个函数的代码，那链接器就会从 归档文件，提取这20个函数的代码放到 hello 文件。

静态链接并不是会把静态库的内容全部复制，只有用到才会复制。链接器能通过一些符号跟数据结构知道 hello 用了 libc.a 里面的哪些函数。按需提取。

------

通常 静态链接 C 运行时库 是为了解决兼容性问题。因为 Linux 系统各个版本的 C运行时库版本 可能不一样。

解决不同版本的问题，还有另一个技巧，不使用静态链接，直接把 libc.so 复制到运行程序当前目录，跟程序一起发布。

然后再在 `/etc/ld.so.conf.d/` ，添加相关配置，再执行 `sudo ldconfig` 即可。因为 ld.so.conf.d 里面的路径是优先于 `/lib` 跟 `/usr/lib` 目录的。

在 Windows 环境，不需要 `ldconfig`，动态库会默认从当前目录优选查找。

------

 参考资料：

1，[《gcc/g++静态链接和动态链接解决glibc版本不兼容的问题》](https://blog.csdn.net/lianshaohua/article/details/82143337)



