# Qt使用FFmpeg的动态库—FFmpeg API教程

C/C++ 项目有两种编译方式，MinGW  跟 MSVC，如下：

1，《[MinGW编译静态库](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-static.html)》

2，《[MSVC编译静态库](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc-static.html)》

但是 MinGW 编译的静态库，是不能给 MSVC 的链接器使用的，虽然两者编译器生成的 .o 跟 .obj 目标文件是 ABI 兼容的，但是他们的链接器不通用。

而 静态库 实际上就是把 目标文件 打包在一起，所以 MinGW 编译的静态库，只能给 MinGW 编译方式的项目使用。MSVC 同理。

------

下面来讲一下 qt creator 如何引入 用 MinGW 方式编译出来的 FFmpeg 动态库。

MinGW 编译 FFmpeg 请参考 《[用msys2与mingw编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw.html)》，编译完成之后，目录如下：

![qt-shared-1-1](qt-shared\qt-shared-1-1.png)

上图中，这些 lib 并不是静态库，而是**动态库的导入库**。而 dll 文件就是动态库，使用 FFmpeg 的库的时候，还需要**按需**引入一些头文件，哪些头文件后面会讲到。

实际上 ffmpeg-n4.4.1-mingw 目录，是有两种导入库的，一种是 xxx.lib ，一种是 xxx.dll.a ，如下：

![qt-static-1-1-2](qt-static\qt-static-1-1-2.png)

可以这么理解这两种导入库，lib 是给 msvc 链接器用的，.a 的导入库是给 MinGW 链接器用的。MinGW  跟 MSVC 对于格式的处理有点不一样，所以需要分开。

这里提醒一下，虽然 dll 是用 MinGW 的 gcc 编译出来的，但是 dll 也可以给 MSVC 用的。他们是兼容的。

不过，我实际测试后发现，qt creator 的 MinGW 可以用 lib 导入库，也可以用 .a 导入库，都可以。但是 qt creator 的 MSVC 只能用 lib 导入库 ，用  .a 导入库来定位就会报错。

------

打开 Qt Creator，点击菜单栏的 New File or Project，选择 Non-Qt Project ，选择 Plain C Application。

![qt-shared-1-2](qt-shared\qt-shared-1-2.png)

项目名称填 **ffmpeg-shared-libs-use**，编译工具选 qmake，如下：

![qt-shared-1-3](qt-shared\qt-shared-1-3.png)

勾选编译环境 Kit ，**Qt 5.15.2 MinGw 64-bit**

![qt-shared-1-4](qt-shared\qt-shared-1-4.png)

------

然后把 `C:\msys64\home\loken\ffmpeg\build64` 这个目录拷贝到 **ffmpeg-qt-version** 项目目录下，如下：

![qt-shared-1-5](qt-shared\qt-shared-1-5.png)

![qt-shared-1-6](qt-shared\qt-shared-1-6.png)

然后 修改 ffmpeg-shared-libs-use.pro 的内容，修改为以下内容：

```
TEMPLATE = app
CONFIG += console c++11
CONFIG -= app_bundle
CONFIG -= qt
#定义 DLL 的导入库的类型
#DLL_IMPORT_TYPE = msvc
DLL_IMPORT_TYPE = mingw

#HEADERS += xxx.h
SOURCES += main.c

contains(QT_ARCH, i386) {
    message("32-bit")
}

contains(QT_ARCH, x86_64) {
    message("64-bit")
    message($$DLL_IMPORT_TYPE)

    contains(DLL_IMPORT_TYPE, mingw) {
        INCLUDEPATH += $$PWD/build64/ffmpeg-n4.4.1-mingw/include
        LIBS += $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libavformat.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libavcodec.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libavdevice.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libavfilter.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libavutil.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libpostproc.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libswresample.dll.a \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/lib/libswscale.dll.a
    }

    contains(DLL_IMPORT_TYPE, msvc) {
        INCLUDEPATH += $$PWD/build64/ffmpeg-n4.4.1-mingw/include
        LIBS += $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/avcodec.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/avdevice.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/avfilter.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/avformat.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/avutil.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/postproc.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/swresample.lib \
            $$PWD/build64/ffmpeg-n4.4.1-mingw/bin/swscale.lib
    }
}
```

提示：gitbook 生成 html 的时候 改乱了 上面的代码，请按照下图进行修改：

![qt-shared-1-6-2](qt-shared\qt-shared-1-6-2.png)

然后把 main.c 的内容也修改一下，创建项目的时候只是输出了一个 hello world，我们现在在 main.c 里面调一下 FFmpeg 的一些 API 函数，如下：

```
#include <stdio.h>
#include "libavcodec/avcodec.h"
#include "libavdevice/avdevice.h"
#include "libavfilter/avfilter.h"
#include "libavformat/avformat.h"
int main()
{
    printf("avcodec version is %u \n",avcodec_version());
    printf("avdevice version is %u \n",avdevice_version());
    printf("avfilter version is %u \n",avfilter_version());
    printf("avformat version is %u \n",avformat_version());
    return 0;
}
```

上面的代码打印了 4 个库的版本号，我为什么知道有这 4个函数可以用，可以用以下命令查看 dll 提供了哪些函数可以调用。

```
dumpbin /EXPORTS ./avcodec-58.dll > avcodec-58.txt
```

![qt-shared-1-7](qt-shared\qt-shared-1-7.png)

然后去 FFmpeg-n4.4.1 的源码里面搜这个函数的定义，有哪些参数，返回值就行了。一般在 头文件里面，所以我上面引入了 4个头文件。

![qt-shared-1-8](qt-shared\qt-shared-1-8.png)

也可以查官网的文档手册来看 API 函数的调用。

------

上面这样改好之后，点 qt creator 编译运行的时候会报找不到 一些dll，这时候需要把 这些缺失的 dll 复制到 exe 的同目录下，因为 我们的 FFmpeg 动态库在 MinGW 环境下编译的，所以依赖一些 MinGW的库，也需要把这些库复制过来，例如 `libiconv-2.dll` ，`liblzma-5.dll`  等等。

这些库一定要复制对，不要把 32位的 复制过来，推荐 Everything 搜索，如下：

![qt-shared-1-9](qt-shared\qt-shared-1-9.png)

复制完之后如下：

![qt-shared-2-1](qt-shared\qt-shared-2-1.png)

这样，再次运行，就可以正常打印出 信息，如下：

![qt-shared-2-2](qt-shared\qt-shared-2-2.png)

------

下面再使用 qt creator 的 MSVC 来编译 main.c 看看，如下：

![qt-shared-2-3](qt-shared\qt-shared-2-3.png)

同时要修改一下 DLL_IMPORT_TYPE 变量，设置为 msvc，这样链接的时候就会使用 lib 后缀的导入库，因为我 pro 文件里面用了这个 DLL_IMPORT_TYPE  变量来执行不同的分支代码。

![qt-shared-2-4](qt-shared\qt-shared-2-4.png)

------

前面 引入的 都是 msys2 + mingw 编译出来 FFmpeg 动态库。

qt creator 如何 引入 msys2 + msvc 编译出来 FFmpeg 动态库 原理也是类似的，这里就不讲解了。
