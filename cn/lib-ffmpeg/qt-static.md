# Qt使用FFmpeg的静态库—FFmpeg API教程

本文讲解 如何在 qt creator 引入 FFmpeg 的静态库，静态库是通过 msys2 + msvc 编译出来的，参考 [《用msys2与msvc编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)，只要在 configure 的时候不使用 `--enable-shared` 就会编译出静态库。FFmpeg 的静态库跟动态库只能二选一，如果都想要，需要编译两次。

编译完成之后，静态库如下：

![qt-static-1-1](qt-static\qt-static-1-1.png)

这些 .a 文件实际上就是 msvc 的 lib 静态库，只是后缀名不一样。

这里提醒一下，这些静态库 只是 FFmpeg 本身代码的目标文件的集合，外部的扩展静态库没有在这些 .a 文件里面，例如 SDL 的静态库 不会在这些 .a 文件里面。

可以通过 上图 的 **pkgconfig** 目录的文件看到 .a 文件依赖的其他外部库，如下：

![qt-static-1-2](qt-static\qt-static-1-2.png)



------

还是在 之前的 ffmpeg-shared-libs-use 项目上讲解，现在把 ffmpeg-4.4-msvc 这个目录 复制到 build64 目录下，如下：

![qt-static-1-3](qt-static\qt-static-1-3.png)

![qt-static-1-4](qt-static\qt-static-1-4.png)

修改一下 **ffmpeg-shared-libs-use.pro** 文件，内容如下：

```
TEMPLATE = app
CONFIG += console c++11
CONFIG -= app_bundle
CONFIG -= qt
#定义 DLL 的导入库的类型
#DLL_IMPORT_TYPE = msvc
DLL_IMPORT_TYPE = msvc-static

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

  	contains(DLL_IMPORT_TYPE, msvc-static) {
        INCLUDEPATH += $$PWD/build64/ffmepg-4.4-msvc/include
        LIBS += $$PWD/build64/ffmepg-4.4-msvc/lib/libavcodec.a mfplat.lib mfuuid.lib ole32.lib strmiids.lib ole32.lib user32.lib \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libavdevice.a psapi.lib ole32.lib strmiids.lib uuid.lib oleaut32.lib shlwapi.lib gdi32.lib vfw32.lib \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libavfilter.a \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libavformat.a secur32.lib ws2_32.lib \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libavutil.a user32.lib bcrypt.lib \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libpostproc.a \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libswresample.a \
            $$PWD/build64/ffmepg-4.4-msvc/lib/libswscale.a
    }

}
```

DLL_IMPORT_TYPE 这个变量名起得不太好，不要介意，虽然是本文是静态库，但是我也用了这个变量。

静态库编译，会发现引入了很多其他的 lib，那些 lib 基本都是导入库，不是静态库。**上面这些 lib 是我从 pc 文件扣出来**，如果不写上，编译就会报错，因为 .a 文件只是 FFmpeg 自己的目标文件，不包含他所依赖的外部库的代码。

------

现在 用 qt creator 的 MSVC 来编译项目，正常打印版号，如下：

![qt-static-1-5](qt-static\qt-static-1-5.png)

我们再来看一下 ffmpeg-shared-libs-use.exe 的依赖，如下：

![qt-static-1-6](qt-static\qt-static-1-6.png)

![qt-static-1-7](qt-static\qt-static-1-7.png)

现在不需要拷贝之前那些 avcodec-58.dll 之类的 动态库过来都没事，因为 这些代码已经静态编译进去  ffmpeg-shared-libs-use.exe 里面了，会发现这个 exe 比之前大了很多。

 ffmpeg-shared-libs-use.exe 这个名字没取好，实际上现在已经没有使用 FFmpeg 动态库了。

------

到此，qt creator 使用 msys2 + msvc 编译的 FFmpeg 静态库已经讲解完毕。

qt creator 如何使用 msys2 + mingw 编译的 FFmpeg 静态库 也是类似的原理，不过要注意，如果 msys2 里面用的 mingw 编译静态库，那 qt creator 也要使用 MinGW 的编译套件。要不会不兼容。
