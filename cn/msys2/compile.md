# MSYS2编译C/C++程序

<div id="meta-description---">实际上使用 MSYS2 编译 C/C++ 跟用 MinGW 编译 C/C++ 是一样的。msys2 里面也是使用的 MinGW，不过 msys2 配套的工具会更丰富一点。
而且 msys2 可以轻松继承 vs2019 的环境变量，使用 cl.exe 来编译。这样就能使用 shell 跟 Makefile 来调用 MSVC 编译项目。
</div>

实际上使用 [MSYS2](https://www.msys2.org/) 编译 C/C++ 跟用 MinGW 编译 C/C++ 是一样的。msys2 里面也是使用的 MinGW，不过 msys2 配套的工具会更丰富一点。

而且 msys2 可以轻松继承 vs2019 的环境变量，使用 cl.exe 来编译。这样就能使用 shell 跟 Makefile 来调用 MSVC 编译项目。

看一下 msys64 里面的文件，发现他里面有个 mingw64 的文件夹，如下：

![msys2-1-1](msys2\msys2-1-1.png)

这些命令，就是从 MinGW 项目拷贝过来的。



------

下面演示一下如何在 msys2 环境下 编译 C/C++ 项目。

打开普通的命令行窗口，进入 C:\msys64 目录，执行 以下命令：

```
cd C:\msys64
.\msys2_shell.cmd -mingw32
```

![msys2-1-2](msys2\msys2-1-2.png)

就会弹出一个新的窗口，这个新的命令行窗口 就是 msys2 的运行环境，这个命令行窗口是使用 [mintty](https://mintty.github.io/) 实现的。

上面命令参数 `-mingw32` 是指使用 32 的的 mingw，也就是 gcc 编译出来的 exe 会是 32 位的。 `-mingw64` 就是 64 位。

------

我在我的用户目录下 创建一个 c-single 目录，然后创建一个 hello.c ，演示一下 用 在 msys2 的环境用 gcc。

![msys2-1-3](msys2\msys2-1-3.png)

 hello.c 代码如下：

```
#include <stdio.h>
int main()
{
    printf("hello ffmpeg \r\n");
    return 0;
}
```

编译命令如下：

```
gcc -o hello.exe hello.c
```

上面我简写了，把编译链接一条命令处理完。

![msys2-1-4](msys2\msys2-1-4.png)

上图中，我用了 windows 的 dumpbin.exe ，为什么能用这个命令，是因为我 修改了 msys2_shell.cmd 里面的 MSYS2_PATH_TYPE ，如下：

![msys2-1-5](msys2\msys2-1-5.png)

这样，执行 `.\msys2_shell.cmd -mingw32` 命令的时候 就会继承 当前窗口的环境变量，这样 之前 窗口能用的命令，我在 msys2 里面都能使用。

也可以使用参数 -use-full-path 来设置 MSYS2_PATH_TYPE 为 inherit。

借助继承当前窗口环境变量的这个原理，我们如果用 x64 Native Tools Command Prompt for VS 2019 窗口执行 msys2_shell.cmd，那在 msys2 里面就可以调 cl.exe 跟 link.exe 了，如下：

![msys2-1-6](msys2\msys2-1-6.png)

上图中，我 没有指定 `-mingw32` ，因为我不需要用到 gcc，如果没指定 `-mingw32`，默认 gcc 是 64位。

然后，因为我是在 64位的 vs2019 窗口打开的 msys2 ，所以 cl.exe 也是 64位的。如果需要 在 msys2 中使用 32位的 msvc ，需要用  **x86** Native Tools Command Prompt for VS 2019 来执行 msys2_shell.cmd，这样 msys2 继承的就是 32位的 MSVC。

**msys2 的特性是继承，你当前窗口是什么环境，他继承的就是什么环境。**

------

cl.exe 可以用，那编译 hello.c 肯定没问题，但是这样没意思，我们写一个 makefile 来编译 hello.c 。

由于 vs2019 的link.exe 跟 MinGW 的 link.exe 会冲突，如下：

![msys2-1-7](msys2\msys2-1-7.png)

上图中， /usr/bin/link.exe 比 vs2019 的link.exe 优先级更高，所以直接 用 link.exe 命令会有问题，可以直接写 vs2019 的绝对路径，但是比较长，所以我新建了一个 mslink 脚本，这个脚本会优选使用 跟 cl.exe 同目录的 link.exe，内容如下：

```
#!/bin/sh
LINK_EXE_PATH=$(dirname "$(command -v cl)")/link
if [ -x "$LINK_EXE_PATH" ]; then
    "$LINK_EXE_PATH" $@
else
    link.exe $@
fi
exit $?
```

然后 执行以下命令 赋予 mslink 执行的权限：

```
chmod 755 mslink
```

然后就可以新建一个 makefile 文件，内容如下

```
hello2.exe: hello.obj
	./mslink -out:hello2.exe hello.obj
hello.obj: hello.c
	cl.exe -c hello.c
```

然后 执行 make 命令，即可编译出来 hello2.exe，如下：

![msys2-1-8](msys2\msys2-1-8.png)

![msys2-1-9](msys2\msys2-1-9.png)

后面 FFmpeg 的 MSVC 版本也是通过这种方式来编译出来的，在 msys2 里面用 makefile 跟 cl.exe 。
