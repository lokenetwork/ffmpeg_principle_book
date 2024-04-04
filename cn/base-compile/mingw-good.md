# MinGW的优势—编译链接基础知识

<div id="meta-description---">MinGW的优势</div>

之前的文章已经演示了 用 MinGW gcc 编译出 exe 文件，打印一个 hello。

但是 用 MSVC 一样能编译出 exe，打印一个 hello。那用 MinGW 的 gcc 意义何在。

------

点击 C:\MinGW\bin\mingw-get.exe ，勾选下面的 mingw32-pthreads-w32dev 安装包，安装到 MinGW。

![mingw-good-1-1](mingw-good\mingw-good-1-1.png)

安装完成之后，在 include 目录下就多了一个 pthread.h 头文件，如下：

![mingw-good-1-2](mingw-good\mingw-good-1-2.png)



------

在 C:\MinGW\projects 下面新建一个项目 **zeus-pthread** ，新建一个文件 zeus-pthread.c ，代码如下：

```
#include<stdio.h>
#include<stdlib.h>
#include<pthread.h>

static void * pthread(void *arg)
{
    printf("hello ffmpeg 1\n");

    return NULL;
}

int main(int agrc,char* argv[])
{
    pthread_t tidp;

    /* 创建线程pthread */
    if ((pthread_create(&tidp, NULL, pthread, NULL)) == -1)
    {
        printf("create error!\n");
        return 1;
    }

    /* 等待线程pthread释放 */
    if (pthread_join(tidp, NULL))
    {
        printf("thread is not exit...\n");
        return -2;
    }
    printf("hello ffmpeg 2\n");

    return 0;
}
```

执行以下命令开始编译：

```
cd C:\MinGW\bin
.\gcc.exe -g3 -c -o C:\MinGW\projects\zeus-pthread\zeus-pthread.o C:\MinGW\projects\zeus-pthread\zeus-pthread.c
.\gcc.exe -g3 -o C:\MinGW\projects\zeus-pthread\zeus-pthread.exe C:\MinGW\projects\zeus-pthread\zeus-pthread.o -lpthread
```

![mingw-good-1-3](mingw-good\mingw-good-1-3.png)

我们把 zeus-pthread.exe 移动到 C:\MinGW\bin 目录下，因为依赖一些 DLL。

![mingw-good-1-5](mingw-good\mingw-good-1-5.png)

这完全 跟 在 Linux 使用 gcc 跟多线程函数一样。

我们再用 Dependencies 查看一下 zeus-pthread.exe 的依赖库，如下：

![mingw-good-1-6](mingw-good\mingw-good-1-6.png)

只依赖于两个 dll。Windows 平台原本是没有 `pthread_create` 函数的。

------

不过 MinGW 没有 提供 fork 函数。如果需要更好的移植性，可以使用 cywin，cywin 有 fork 函数。

下面是 官网 介绍的 MinGW 的优势。

------

## Headers, Libraries and Runtime

- More than a million lines of headers are provided, not counting generated ones, and regularly expanded to track new Windows APIs.
- Everything needed for linking and running your code on Windows.
- Winpthreads, a pthreads library for C++11 threading support and simple integration with existing project.
- Winstorecompat, a work-in-progress convenience library that eases conformance with the Windows Store.
- Better-conforming and faster math support compared to VisualStudio's.

## Tools

- gendef: generate Visual Studio .def files from .dll files.
- genidl: generate .idl files from .dll files.
- widl: compile .idl files.
