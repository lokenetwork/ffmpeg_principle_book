# Android使用FFmpeg的API库—FFmpeg API教程

<div id="meta-description---">Android 提供了一套 NDK 工具，来编译使用 C/C++ 代码，本文主要讲解如何把 FFmpeg 的so动态库打包进去 apk 进行使用。so 库完全是由原生的 CPU 指令堆叠起来的，所以运行速度很快</div>

Java 可以通过 JNI 调用原生库中的函数，原生库完全是由原生的 CPU 指令堆叠起来的，所以运行速度很快。大部分的原生库都是用 C/C++ 编译出来的。

因此，Android 里面也能通过 JNI 的方式调用 `so` 动态库，或者 `.a` 静态库里面的函数。JNI 的全称是 Java Native Interface，Java 原生接口。

Android 提供了一套 NDK 工具，来编译使用 C/C++ 代码。有关 NDK，JNI 相关的资料，本文不进行讲解，推荐阅读以下资料。

1，《Java Native Interface: Programmer's Guide and Specification》- Sheng Liang

2，《[Android Studio添加C/C++代码](https://developer.android.com/studio/projects/add-native-code?hl=zh-cn)》

3，《[NDK 使用入门](https://developer.android.com/ndk/guides?hl=zh-cn)》，这是官方文档。

4，《细说Android 4.0 NDK编程》- 王家林

----

掌握 `JNI/NDK` 基本概念之后，现在就来实战巩固一下，引入 ffmpeg 的动态库给之前的 Hello app 使用，相关代码可在 [Github](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/android) 下载。

首先我们需要使用 Android Studio 里面的 NDK 的 clang 编译器来编译 ffmpeg 的代码，编译出来 so 动态库。本文采用的 ffmpeg 版本是 n4.4.1。

如果没有安装 NDK，可以点击 Tools > SDK Manager 进行安装，如下：

![1-1](android\1-1.png)

安装完成之后，会在 SDK 的安装目录上看到这个 `clang.exe` 编译器，如下：

![1-2](android\1-2.png)

本文使用的 NDK 版本是 23.1，在进行 NDK 开发的时候，谷歌推荐使用 clang 编译器来编译 C/C++ 代码，而不是 gcc 或者 msvc。

---

由于我们是在 Windows10 系统上编译 C/C++ 代码，生成手机上运行的 CPU 指令，这是交叉编译，也叫跨平台编译，所以需要指定 `--target`，指定生成那种 CPU 架构的指令集，同时也要指定 Android 的 API 级别。

手机的 CPU 架构可以用命令查看，先通过 `adb` 链接上手机，然后执行 `cat /proc/cpuinfo` 命令，如下：

```
cd C:\Users\loken\AppData\Local\Android\Sdk\platform-tools
adb.exe shell
cat /proc/cpuinfo
```

![1-4](android\1-4.png)

可以看到，CPU 架构是 8 ，那 8 代表什么呢？ 8 代表这个 CPU 是 ARMv8 的架构，暂时可以把 aarch64 跟 ARMv8 看成是一个东西。

我的手机是 Redmi 5 Plus，Android 版本 8.1.0，对应的 API 级别是 27。读者可以在《[SDK 平台版本说明](https://developer.android.com/studio/releases/platforms?hl=zh-cn)》查阅自己手机的 Android 版本对应的 API 级别。**因此本文的设备是 aarch64 + API27**。

Android Studio 已经提前弄好一些 shell 跟 batch 脚本给我们用，如下：

cmd 后缀是用在 windows 命令行的，没有 cmd 的是 shell 脚本，用在 Linux 的命令行。

![1-5](android\1-5.png)

这个 `aarch64-linux-android27-clang` 其实就是一个 shell 脚本，里面也是调的 `clang.exe` 编译器，只是这个 shell 脚本预设了一些参数，它的部分内容如下：

```
`dirname $0`/clang.exe --target=aarch64-linux-android27 "$@"
```

我们不需要用到这个 shell 脚本，只是参考一下他的 `--target` 参数。

---

下面我们打开 msys2 的 shell 窗口，这里直接使用下面的命令打开即可，不需要继承 vs2019 的环境变量，因为不需要用到 msvc。

```
cd C:\msys64
.\msys2_shell.cmd -mingw64
```

在 msys2 环境下，安装一些 必要的软件：

```
# 刷新软件包数据
pacman -Sy  
# 安装mingw-w64。
pacman -S mingw-w64-x86_64-toolchain
pacman -S git
pacman -S make
pacman -S automake 
pacman -S autoconf
pacman -S perl
pacman -S mingw-w64-x86_64-SDL2
pacman -S libtool
pacman -S mingw-w64-x86_64-cmake
pacman -S pkg-config 
pacman -S yasm
pacman -S diffutils
# 编译x264 需要 nasm
pacman -S nasm
```

---

然后上 Github 下载 [FFmpeg-n4.4.1.zip](https://github.com/FFmpeg/FFmpeg/archive/refs/tags/n4.4.1.zip) 代码，放到 下图中的目录，这样 msys2 环境也能找到。

![msys2-mingw-1-1](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw/msys2-mingw-1-1.png)

![msys2-mingw-1-2](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-mingw/msys2-mingw-1-2.png)

------

然后我们需要设置一下 PATH 环境变量，让 `clang.exe` 编译器能被找到，命令如下：

```
export PATH=$PATH:/C/Users/loken/AppData/Local/Android/Sdk/ndk/23.1.7779620/toolchains/llvm/prebuilt/windows-x86_64/bin
```

![1-6](android\1-6.jpg)

进入 FFmpeg-n4.4.1 目录，开始编译，命令如下：

```
cd /home/loken/ffmpeg/ffmpeg-n4.4.1

./configure \
--prefix=/home/loken/ffmpeg/build64/ffmpeg-n4.4.1-android \
--target-os=android \
--arch=arm64  \
--cpu=armv8-a \
--cc=clang \
--cxx=clang++ \
--strip=llvm-strip \
--extra-cflags="--target=aarch64-linux-android27" \
--extra-ldflags="--target=aarch64-linux-android27" \
--extra-cflags="-Os -fpic -march=armv8-a" \
--disable-asm \
--disable-strip \
--disable-doc \
--disable-ffplay \
--disable-ffprobe \
--disable-symver \
--disable-ffmpeg \
--enable-neon \
--enable-cross-compile \
--enable-shared

make -j8
make install
```

上面的命令有一个重点，就是无论是编译阶段，还是链接阶段，都需要加上 `--target=aarch64-linux-android27`，要不会报错的。

网上有些文章说要指定 `--sysroot` 的路径，但是好像我不指定也没问题。

上面的命令执行完之后，ffmpeg-n4.4.1-android 目录的结构如下：

![1-7](android\1-7.png)

---

so 动态库编译出来之后，就可以在 Android Studio 里面使用了。

先在原来的 Hello 项目里面 main 目录下创建一个 jinLibs 文件夹，然后再创建一个 arm64-v8a 文件夹，如下： 

![1-8](android\1-8.png)



然后把之前生成的 so 库全部拷贝到 arm64-v8a 目录，如下：

![1-9](android\1-9.png)

要使用这些动态库里面的函数，还需要有头文件，头文件我们不需要一个一个拷贝，之前 `make install` 的时候，已经把所需要的头文件拷贝到 `build64/ffmpeg-n4.4.1-android` 的 include 目录下，如下：

![2-1](android\2-1.png)

现在我们在把 `build64/ffmpeg-n4.4.1-android` 的 include 目录拷贝到 Hello app 项目的 cpp 目录下，通常 Android 项目是没有 cpp 目录的，所以我们需要右键 app 目录，点击 Add C++ to Module，如下：

![2-3](android\2-3.png)

![2-4](android\2-4.png)

这样操作，Android Studio 就会帮我们生成 cpp 目录，以及相关的 `CMakeLists.txt` 跟 `hello.cpp` 。

提醒：这些 cpp 目录以及文件，是可以自己手动添加的，但是我试了一下，手动添加完之后，Android Studio 没有加载出来，可能是有个配置文件负责管理的，所以保险起见，还是用 Add C++ to Module 这个功能。

---

现在我们在把 `build64/ffmpeg-n4.4.1-android` 的 include 目录拷贝到 Hello app 项目的 cpp 目录，如下：

![2-5](android\2-5.png)

现在我们需要对 `CMakeLists.txt` 做一下修改，把 FFmpeg 的 so 库引入进来，重点的修改如下：

![2-6](android\2-6.png)

![2-7](android\2-7.png)

然后再修改一下 `hello.cpp` 的代码，调用一下 so 库里面的一个版本函数，代码如下：

```
#include <jni.h>
#include <string>
#include <unistd.h>

extern "C" {
    #include <libavcodec/avcodec.h>
    #include <libavformat/avformat.h>
    #include <libavfilter/avfilter.h>
    #include <libavcodec/jni.h>

    JNIEXPORT jstring JNICALL
    Java_com_example_hello_FirstActivity_ffmpegVersion(JNIEnv *env, jobject  /* this */) {
        unsigned ver = avformat_version();
        char info[40000] = {0};
        sprintf(info, "avformat_version %u:", ver);
        return env->NewStringUTF(info);
    }
}
```

然后我们就可以在 FirstActivity 里面调用这个 `ffmpegVersion()` 函数了，如下：

![2-8](android\2-8.png)

上图中，我把 avformat 库的版本号加在后面了，然后直接编译项目把 apk 安装到手机，整个过程如果没有报错，就会生成 `app-debug.apk` 文件，如下：

![2-9](android\2-9.png)

Android Studio 提供了一个 `apkanalyzer.exe` 工具来分析 apk 文件，我们可以分析一下 apk，如下：

![2-9-2](android\2-9-2.png)

![2-9-3](android\2-9-3.png)

可以看到，so 库已经打包进去 apk 里面了。

手机上的运行效果如下：

<img src="android\2-9-4.png" alt="2-9-4" style="zoom:33%;" />

Jni 完整的项目代码可以点击 [helloJni](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/android/helloJni) 进行下载。

---

官网文档的 [两篇文章](https://developer.android.com/ndk/guides/android_mk?hl=zh-cn) 说要用 Android.mk 跟 Application.mk 来引入外部的库，但是我没看懂这两篇文章，貌似不创建这两个文件也可以。

---

本文只是做了一些简单的介绍，把 FFmpeg 的库编译进去 apk，然后调用一个函数。关于 NDK开发，料推荐阅读以下书籍 深入学习

1，《Android C++高级编程-使用NDK》

2，《细说Android 4.0 NDK编程》

---

结尾，分享一些本人的学习技巧，本文是参考《[Android FFmpeg 编译和集成](https://cloud.tencent.com/developer/article/1773965)》一文进行写作的，这篇文章里面创建了一个 `build_android_clang.sh` 脚本，然后定义一些变量传递进去，如下：

![3-1](android\3-1.png)

我能看懂大部分的 shell 以及 batch（cmd）代码，我看了一下这个 shell，为了降低本文的复杂度，所以我没有创建新的 shell 脚本，而是直接把参数写在 `configure` 上了。

再回到之前的 Android Studio 提供的 `aarch64-linux-android27-clang` 脚本。

```
`dirname $0`/clang.exe --target=aarch64-linux-android27 "$@"
```

 这个脚本实际上就是给 `clang.exe` 编译器定义一个别名，这个别名会附加上 `--target` 的参数。

由于 `clang.exe` 会被调用两次，一次是编译的时候，一次是链接的时候，所以当把 参数往 `configure` 上丢的时候，需要丢两次，所以就有了下面的代码：

```
./configure \
...省略...
--extra-cflags="--target=aarch64-linux-android27" \
--extra-ldflags="--target=aarch64-linux-android27" \
...省略...
```

上面的命令，`--extra-cflags` 是往编译阶段传递参数，`--extra-ldflags` 是往链接阶段传递参数。

本书的《[FFmpeg编译过程分析](https://ffmpeg.xianwaizhiyin.net/build-ffmpeg/)》一章专门分析 `configure` 的代码。

---

然后，我没有使用 Ubuntu 或者 Mac 来编译 FFmpeg 的动态库，虽然也可以。我平时是在 Windows 上开发的，所以直接就采用 `msys2` 了，在前面的文章《[用msys2与msvc编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)》里，msys2 可以调 vs2019 的 `msvc` 编译器 `cl.exe` ，同理 msys2 肯定也能调 Android Studio 的 NDK 编译器 `clang.exe`，这些东西都是可以举一反三串联起来的。

最后，FFmpeg 官方的 `configure` 脚本非常值得称赞，这个脚本使得 很多选项可以配置，例如可以指定编译器，链接器，strip ，等等。因此 `configure` 的通用性非常强，在各种平台都能跑。Windows 可以用 msys2 来执行 shell 脚本，其他类 Linux 平台本身就支持 shell。



