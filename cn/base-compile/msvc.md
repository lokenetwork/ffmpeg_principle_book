# MSVC编译环境介绍—编译链接基础知识

<div id="meta-description---">MSVC 是 windows 比较早期的编译工具链，这个工具链主要有 两个软件， cl.exe （编译器） 跟 link.exe（链接器）。</div>

MSVC 是 windows 比较早期的编译工具链，这个工具链主要有 两个软件， `cl.exe`（编译器） 跟 `link.exe（`链接器）。

这两个 软件 通常集成在 vs2019 里面。点击 vs2019 菜单栏 Tool → Get Toole and Feature ，如下：

![msvc-1-2](msvc\msvc-1-2.png)

安装上 MSVC 之后，需要找到 **x64 Native Tools Command Prompt for VS 2019**，这个是 vs2019 的CMD，用这个 CMD 启动命令行会有 vs2019 的环境变量。

![msvc-1-3](msvc\msvc-1-3.png)

打开后，可以使用 `where` 查询 `cl.exe` 跟 `link.exe` 的位置。如下：

![msvc-1-4](msvc\msvc-1-4.png)

------

下面就来使用一下 `cl` 跟 `link` 来编译 C 代码，其实这两个软件 跟 之前在 Linux 环境用 `gcc` 是类似的。都是单个文件编译，把 C程序 编译成 字节码，然后用链接器把多个字节码文件链接在一起。

还是以 之前的单文件项目 c-single 为例，`hello.c` 代码如下：

```
#include <stdio.h>
int main()
{
    printf("hello ffmpeg \r\n");
    return 0;
}
```

我在 D 盘 创建了 c-single 文件夹，把 `hello.c` 放在里面。然后在  vs2019 的CMD 里面使用 `cl.exe` 编译这个文件。如下：

```
cl.exe /c /Fohello.obj hello.c
```

![msvc-1-5](msvc\msvc-1-5.png)

上面命令中的 `/c` 跟 linux gcc 的 `-c`  一样，都是只编译，不进行链接操作。cl.exe 的选项 习惯都是 `/` 开头的，而 gcc 的选项是 以 `-` 开头。

目前 `cl.exe` 也支持 `-c` 的写法。

`/Fo` 这个选项是指定输出的 目标文件名。

Windows 平台编译过程通常会遇到 有 3种 后缀的文件。

1，`.obj` 后缀，这是目标文件。

2，`.lib` 后缀，这个是静态库，或者是 动态库的 import 库。

3，`.dll` 后缀，动态库。

Windows 的动态库跟 Linux 的动态库不太一样，Linux 是把所有信息都放在 so 文件里面，而 Windows 把 import 导入信息单独放在 lib 文件来 让编译阶段能找到关联关系，实现隐式调用动态库。

因此，Windows 的dll 动态库，在编译以及链接阶段 都不需要使用，只有程序运行的时候才需要。编译以及链接阶段需要用到动态库的 import 库即可。

而 Linux 的 so 动态库，我们在编译的时候要指定so文件，运行的时候也要指定so文件，因为他所有信息都放在一起了。

------

无论是 `.obj`，`.lib`，`.dll`  其实他们的文件格式都是 COFF 格式。而 Windows 的 PE 格式 跟 Linux 的 ELF 格式 都是基于 COFF 格式发展来的。

exe 文件就是 PE 格式。PE 跟 ELF 有很多相似之处，都是基于段的结构。

------

我们已经编译出来 `hello.obj` 目标文件了，怎么查看里面的内容呢？Windows 提供了 跟 Linux 的 objdump 类似的工具 **dumpbin**。

我们查看一下 `hello.obj` 里面的汇编代码，命令如下：

```
dumpbin /DISASM hello.obj
```

![msvc-1-6](msvc\msvc-1-6.png)

从上图可以看到，`printf` 的地址也是没有修正的。

不过 `dumpbin` 是命令行工具，对于初学者还是不太优好，这里推荐 [PEview](http://wjradburn.com/software/) 跟 HxD 来查看 PE/COFF 文件。

![msvc-1-7](msvc\msvc-1-7.png)

------

回到 本文主题，`hello.obj` 已经出来了，下面就到 链接阶段，因为 `hello.c` 是 C 程序，所以跟 Linux 类似，需要链接 C运行时库。命令如下：

```
link.exe /DEBUG /OUT:hello.exe hello.obj
```

`/OUT` 是指定输出文件名。

![msvc-1-8](msvc\msvc-1-8.png)

我们想知道 这个 `hello.exe` 链接的 C运行时是动态的还是静态的，可以使用 dumpbin 命令来查看，如下：

提醒： [Dependencies](https://github.com/lucasg/Dependencies) 工具查看动态库依赖不太准，还是用 dumpbin 靠谱。

```
dumpbin /dependents hello.exe
```

![msvc-1-9](msvc\msvc-1-9.png)

在 Windows 平台， C运行时库是 `libcmt.lib`（静态库） 跟 `msvcrt.dll` （动态库），因为上图没出现 `msvcrt.dll` ，所以 `hello.exe` 默认是静态链接 C运行时库。

------

那么有什么方法 可以让 `hello.exe` 动态链接 C运行时库呢？使用 `/MD` 选项即可。如下：

```
cl.exe /c /MD /Fohello-md.obj hello.c
link.exe /DEBUG /OUT:hello-md.exe hello-md.obj
dumpbin /dependents hello-md.exe
```

![msvc-1-9-1](msvc\msvc-1-9-1.png)

**Windows 指定 动态还是静态，是在编译阶段指定的**，编译阶段指定之后会保存 链接信息进 obj 文件的 `.drectv` 段，然后 链接阶段就会取这些信息。如下：

![msvc-1-10](msvc\msvc-1-10.png)

如果要静态链接 C运行时库，可以使用 `/MT` ，这个是默认值。

------

扩展知识：`hello.c` 的代码其实也是跨平台的，linux 跟 windows 都可以编译，不需要改动源代码。大家可能经常听说 C++ 的跨平台比较好，但实际上，C语言的很多语法函数，目前也是可以跨平台使用的。

任何一门技术，刚出来的时候，都是混乱的，包括C语言。20世纪70年代C语言变得非常流行时，许多大学、公司和组织都自发地编写自己的C语言变种和基础函数库，因此当到了80年代时，C语言已经出现了大量的变种和多种不同的基础函数库，这对代码迁移等方面造成了巨大的障碍。

因此 美国国家标准协会 成立了一个委员会，建立了 ANSI C 标准，也就是后来的 C89 标准。标准定义了 C运行时库的一些行为。

`printf` 这个函数 是C运行时库里面的函数，C 运行时库屏蔽了不同的 系统实现，提供统一的函数给上层用，例如 windows 下的 `printf` 的内部实现 其实是跟 Linux 的 `printf` 不太一样的。windows 调的是 win32 API 函数。

参考资料：

1，[《link 链接器选项》](https://docs.microsoft.com/zh-cn/cpp/build/reference/linker-options?view=msvc-170)
