# Linux环境编译多个C程序文件—编译链接基础知识

<div id="meta-description---">Linux环境编译多个C程序文件</div>

本文使用的项目 是 [universe](https://pan.baidu.com/s/1_reA14rpTZ6PTwY7Qc_q2Q ) 项目，宇宙的意思，提取码：mku9 。请下载后放到 Document 目录，项目结构如下：

![linux-c-multiple-1-1](linux-c-multiple\linux-c-multiple-1-1.png)

这个项目是这样的，zeus 是 宙斯的意思，他可以控制 3 个星球 ，太阳，月亮，地球 旋转。



------

大部分 C/C++ 开源项目，程序文件都有几百个，甚至几千几万个文件，但是不要被多文件吓到。**gcc 编译的时候实际上都是单文件编译**，然后链接阶段才是多文件，整个流程图如下：

![linux-c-multiple-1-2](linux-c-multiple\linux-c-multiple-1-2.png)



本项目的编译命令如下：

```
gcc -c -o zeus.o zeus.c
gcc -c -o sun.o sun.c
gcc -c -o moon.o moon.c
gcc -c -o earth.o earth.c
```

提示：以上命令可以加上 `-v` 或者 `-###` 显示更多的编译信息。

上面的命令，我是先 编译 zeus.c 文件的，虽然这个 zeus.c 文件依赖 sun.c，moon.c 跟  earth.c  3 个文件，但是 编译阶段不需要知道这 3 个 C 文件的存在，只需要在代码 里 include 一下他们的头文件，知道函数的定义即可。

**编译阶段不需要知道 依赖函数的具体实现。**

编译完成之后，执行以下命令开始链接：

```
gcc -o zeus zeus.o sun.o moon.o earth.o
```

运行情况如下：

![linux-c-multiple-1-3](linux-c-multiple\linux-c-multiple-1-3.png)

至此，一个多文件的C项目，就编译完成。

不过目前的示例过于简单，很多项目的C文件会嵌套使用函数，例如 现在有个新需求， 太阳转的时候 月亮要跟着转。所以我们在 `sun.c` 里面改动一下代码，加上以下代码：

![linux-c-multiple-1-4](linux-c-multiple\linux-c-multiple-1-4.png)

因为 sun.c 调用了 moon_rotate() 函数，所以需要引入 `moon.h` 头文件，但是不要被引入头文件这个操作迷惑，在编译阶段，sun.c 跟 moon.c 依然是没有一丁点关系，依然是独立编译。

在编译阶段，sun.c 里面的 moon_rotate() 会被替换成 call 00 00 00 00 ，占下位置。

在链接阶段，链接器 就会去 扫描 moon.o 文件，找到 `moon_rotate()` 函数的真正地址，然后进行替换，因为  `moon_rotate()` 函数被调用了两次，第一次是在 zeus.c 里面调用，第二次是 在 sun.c 里面调用，所以链接器需要替换两个地方的地址。

再执行之前的编译命令，运行效果如下，月亮旋转了两次。

![linux-c-multiple-1-5](linux-c-multiple\linux-c-multiple-1-5.png)

------

上面的例子还是偏简单，在实际项目中，你可能会遇到一些 Makefile ，gcc 接受多个输入文件的情况，之前说过，gcc 可以接受多个输入文件。

现在 sun.c 依赖 moon.c 的函数，我们试着把这两个文件一起编译，看看会怎样？

```
gcc -c -o sun-moon.o sun.c moon.c
```

![linux-c-multiple-1-6](linux-c-multiple\linux-c-multiple-1-6.png)

从上图可以看到，这条命令立即报错了，因为编译多文件的时候，不能使用 `-c` 参数，`-c` 参数是让 gcc 不要进行链接操作。

咱们 把 `-c` 去掉，再跑一次。

![linux-c-multiple-1-7](linux-c-multiple\linux-c-multiple-1-7.png)

依旧报错，这是因为，如果 gcc 进入 链接阶段 就会去找 main 函数的具体地址，而 main 函数的地址在 zeus.o 里面，zeus.o 没传递给 gcc 命令行参数，所以就报错了。

因此，无论是多庞大的C/C++ 项目，编译阶段，都是单个文件编译的，单个文件里面引用了外部的变量，函数。编译阶段只会把这些引用 替换成一些占位符号，例如 00 00 。

只有在链接阶段，gcc 才可以接受多个 文件作为输入。然后 扫描这些输入文件，找到各个 函数符号地址，然后进行替换。链接器通常会扫两次输入文件，第一次找到所有符号，第二次才开始替换占位符号。

------

通常大型 C/C++ 项目编译问题，主要有以下几种：

1，找不到头文件，错误信息如下：

![linux-c-multiple-1-8](linux-c-multiple\linux-c-multiple-1-8.png)

上面的报错，是因为我把 moon.h 头文件删了。

找不到头文件这种编译错误怎么解决，通常是 头文件的搜索路径里面没有 相关的 moon.h。这时候，需要查看 gcc 命令会从哪些目录查找头文件。

只需加上 -v 参数即可，命令如下：

```
gcc -v -c -o sun.o sun.c
```

![linux-c-multiple-1-9](linux-c-multiple\linux-c-multiple-1-9.png)



扩展知识：Linux 环境的大部分软件的用法，都在 `--help` 里面，**学会看 --help 跟 日志是非常重要的**。而 Windows 的大部分软件，也有 `/?` 来查看帮助，但是微软的文档更全面一些，中文版的文档 微软更全面一些。

上图中日志的 意思是，会在当前目录跟   `/usr/include` ，`/usr/local/include`等等这些目录下搜索 moon.h ，由于这些目录都没有 moon.h 文件，所以编译器就会报错。

因此，如果你的项目编译的时候编译器找不到头文件，而你确确实实有头文件，那就是 头文件的目录不在 编译器搜索路径里面，加上去即可，可以通过 `-I` 参数加上搜索路径。

```
gcc -v -c -o sun.o sun.c -I/home/ubuntu/
```

上面的命令可以编译成功，因为我把 moon.h 放到了 /home/ubuntu/ 目录下面。

------

`#include` 引入头文件有两种写法。

```
#include "moon.h"
#include <moon.h>
```

`<moon.h>` 是只在系统目录下查找，不在当前目录查找，找不到头文件就报错。`"moon.h"` 会在当前目录查找，也会在系统目录查找。

------

2，undefined reference ，错误信息如下：

![linux-c-multiple-1-10](linux-c-multiple\linux-c-multiple-1-10.png)

上面这个错误是因为 我把 moon.o 写漏了，加上即可。

------

3，C/C++ 大型项目还有很多 编译 静态库，动态库的常见问题，下一篇文章再做讲解。

扩展阅读：

1，[《打印头文件搜索路径》](https://wizardforcel.gitbooks.io/100-gcc-tips/content/print-header-search-dir.html)
