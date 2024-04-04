# FFmpeg引入NVIDIA硬件编解码扩展—FFmpeg编译教程

<div id="meta-description---">本文主要介绍如何在 window10 的 msys2 环境下编译 ffmpeg 的 NVIDIA GPU硬件编解码器 h264_nvenc 跟 h264_cuvid。</div>

本文主要介绍 如何在 window10 的环境下编译 ffmpeg 的 NVIDIA GPU硬件编解码器 h264_nvenc 跟 h264_cuvid。

并不是所有的 NVIDIA 显卡都支持 h.264 跟 h.265 编解码的，有些显卡只负责渲染，不支持编解码，例如 GeForce 830M > 945M。

可以通过 [Video Encode and Decode GPU Support Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new#Encoder) 查看各种 NVIDIA 显卡对编解码的支持情况，有些显卡只支持 h.264 ，不支持 h.265。

![nvidia-1-1](nvidia\nvidia-1-1.png)

从上图可以看出，NVIDIA 显卡分为 4 类。

1. Consumer，面向 PC 市场的 GeForce 产品线。
2. Professional，面向工作站市场的 Quadro 产品线。
3. Server（Data Center），针对高性能计算（HPC）的 Tesla 产品线。
4. DGX，以 Volta （伏特）作为微架构的产品线，这条产品线比较新。



------

FFmpeg 源码编译的时候，引入 NVIDIA 的硬件编解码器，其实跟引入 SDL，x264 是类似的，都是链接的时候需要一些 lib 导入库，运行的时候需要一些 dll。

只是 NVIDIA 的编译安装相对麻烦一些，**要特别注意各个部分的版本号是否匹配兼容**。

先查看自己本地 NVIDIA 的驱动版本。点击菜单栏的 `"帮助"` ➔ `"系统信息"` 。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-2.png)

从上图可以看到，本地显卡 GeForce RTX 2060 的**驱动版本号**是 456.71，然后再通过 [CUDA Toolkit and Corresponding Driver Versions](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html) 查看 驱动 456.71 对应的 CUDA Toolkit 的版本。如下：

![nvidia-1-3](nvidia\nvidia-1-2.png)

虽然上图中，CUDA 11.0.1 等等 跟 CUDA 10.1 需要的驱动版本号都是兼容  456.71 的。但是不要选 11.0.1 的版本，因为 FFmpeg 那边可能没更新这么快的。你如果选了 11.0.1，对于初学者可能有点问题，因为 CUDA Toolkit 安装的时候会改变一些 环境变量，一旦装多个版本的 CUDA Toolkit，新手可能不知道如何配置这些环境变量避免混乱。

所以，需要看你本地的驱动版本号，选一个比较旧的，能支持你的驱动的 CUDA Toolkit 来安装即可。

本文选择的是 [CUDA Toolkit 10.1 update2](https://developer.nvidia.com/cuda-10.1-download-archive-update2) ，因为这个版本貌似是个正式版，应该比较稳定。

CUDA Toolkit 历史版本下载：https://developer.nvidia.com/cuda-toolkit-archive

------

这里需要讲解一下 Driver （显卡驱动）跟 CUDA Toolkit （CUDA工具套件）的关系。

显卡驱动是你安装电脑的时候装的。而  CUDA Toolkit 这个安装包，实际上就是把一堆 头文件，lib导入库，dll动态库安装到你电脑，这些东西都是 FFmpeg 编译链接 或者运行的时候需要的。

FFmpeg 进行硬件编解码的时候 是通过 CUDA Toolkit 给的 dll 跟 显卡驱动 通信的。如下图：

![nvidia-1-3](nvidia\nvidia-1-3.png)



上图是这样的， CUDA Toolkit 给的东西，那些头文件，lib导入库，只有在编译 ffmpeg.exe 的时候才需要，一旦编译出来 ffmpeg.exe ，那些 xx.h 跟 xx.lib 都没用了。然后我们只需要把 这4个dll 跟 ffmpeg.exe 一起发布给客户使用即可，客户机不需要安装 CUDA Toolkit。

官网的教程有提及，如下：

**Running a CUDA application requires the system with at least one CUDA capable GPU and a driver that is compatible with the CUDA Toolkit.**

CUDA 的全称是 Compute Unified Device Architecture，计算机统一设备架构，但后来 NVIDIA 取消了个全称。CUDA 就是 CUDA，不需要解释。

------

下面开始安装 CUDA Toolkit 10.1，安装界面如下，**选择自定义安装**。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-3-1.png)

为了更快地安装，我们只需要选择 Development 跟 Runtime ，这两个是 **编译环境** 跟 **运行时**，也就是会把一些 头文件， `lib` 导入库 跟 `dll` 库 安装到我们电脑。

![nvidia-1-4-1](nvidia\nvidia-1-4-1.png)

那些 Nsight 是 NVIDIA 的性能分析工具，咱们初学用不到，选太多东西安装，会经常导致安装失败，如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-3-2.jpg)

这个 CUDA Toolkit 有可能经常安装失败，可以参考 [官网英文教程](https://docs.nvidia.com/video-technologies/video-codec-sdk/ffmpeg-with-nvidia-gpu/#compiling-ffmpeg)。

CUDA Toolkit 安装完毕，在 WinCMD 命令行输入 `nvcc -V`，查看安装是否成功。

![nvidia-1-4](nvidia\nvidia-1-4.png)

我们可以看一下 CUDA Toolkit 安装包装了哪些东西到电脑上，如下图：

![nvidia-1-5](nvidia\nvidia-1-5.png)

![nvidia-1-5](nvidia\nvidia-1-6.png)

![nvidia-1-5](nvidia\nvidia-1-7.png)

从上图可以看出，CUDA Toolkit 就是安装了一些 头文件，lib导入库，DLL动态库。

因此，我们现在需要 把这些 头文件，lib导入库 放到 msys2 的目录，同时做一些相关配置，让 FFmpeg 的编译脚本能找到他们。

在 `/usr/local/include` 新建一个目录 **nv_sdk_10.1** 用来放头文件。

把 `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v10.1\include` 下面所有的头文件复制到 **nv_sdk_10.1** 目录，如下：

![nvidia-1-7-2](nvidia\nvidia-1-7-2.png)

把 `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v10.1\lib\x64` 里面所有的 lib 导入库全部 复制到 `/usr/local/lib/x64`

![nvidia-1-7-3](nvidia\nvidia-1-7-3.png)

这里我就不建 pc 文件来定位这些 头文件跟 lib了，本文后面会介绍一种新的方式，用 **--extra-cflags** 跟 **--extra-ldflags** 来搞。

------

本文 编译的 ffmpeg.exe 是64位，因为 64位才能最大发挥 CUDA Toolkit 的作用。NVIDIA官网教程有讲如何编译 32位，但是我看了下，32位会少很多库，应该会少很多功能，所以本文不讲解32位的编译。

FFmpeg 用 NVIDIA GPU 硬件编解码也是支持 MinGW 的编译方式的，但是 MinGW 的方式我还没实践过，所以不讲解，官网说是支持的。如下：

**FFmpeg with NVIDIA GPU acceleration is supported on all Windows platforms, with compilation through Microsoft Visual Studio 2013 SP2 and above, and MinGW. Depending upon the Visual Studio Version and CUDA SDK version used, the paths specified may have to be changed accordingly.**

所以本文用 msys2 + msvc 来编译 FFmpeg，不熟悉这种编译方式的请看[《用msys2与msvc编译FFmpeg》](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)



------

在开始编译之前，还需要用到 nv-codec-headers 项目的头文件，这里跟 官方教程 [ffmpeg-with-nvidia-gpu](https://docs.nvidia.com/video-technologies/video-codec-sdk/ffmpeg-with-nvidia-gpu) 不太一样，不能直接执行下面命令：

```
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
```

上面命令拉下来的 nv-codec-headers 是master 分支的，需要的显卡驱动至少要 471.41 以上，我们本地显卡驱动是 456.71，因此如果直接用官网的命令会编译失败。

从 `nv-codec-headers` 的 **README** 可以看到 所需显卡驱动版本：

![nvidia-1-8](nvidia\nvidia-1-8.png)

这里提供一个下载 nv-codec-headers 历史版本的地址，[历史版本下载](https://github.com/FFmpeg/nv-codec-headers/releases)。请选择符合自己显卡驱动的版本。本文选择的是 [9.1.23.3](https://github.com/FFmpeg/nv-codec-headers/releases/tag/n9.1.23.3) 版本。

因此，编译 NVIDIA 硬件编解码最重要的地方就是版本匹配，3个组件的版本需要匹配兼容。不匹配 ffmpeg 的 configure 脚本会通不过。

1. 显卡驱动 版本，本文是 456.71
2. CUDA Toolkit 版本，本文是 10.1
3. nv-codec-headers 版本，本文是 9.1.23.3

`nv-codec-headers` 项目里面有个 `makefile` 文件，官网教程提示要执行 `make install`，**但本文不建议使用 make install 来安装 nv-codec-headers，我们手动安装。**

先把 `nv-codec-headers` 里面 `include` 目录的 `ffnvcodec` 文件夹复制到 `/usr/local/include/`，如下：

![nvidia-1-9](nvidia\nvidia-1-9.png)

然后手动创建一个 **ffnvcodec.pc** 文件，内容如下：

```
prefix=/usr/local/include/
includedir=${prefix}

Name: ffnvcodec
Description: FFmpeg version of Nvidia Codec SDK headers
Version: 9.1.23.3
Cflags: -I${includedir}
```

为什么我要手动搞一个 `ffnvcodec.pc`，是因为用 `make install` 生成的 `ffnvcodec.pc` 里面 `Cflags` 的运算结果是 `-IC:/msys64/usr/include`， 用 `cl.exe` 编译的时候，是不能引入 `C:/msys64/usr/include` 里面的 **mingw** 的头文件，会跟 `MSVC` 的头文件同名混乱，导致报错。

如果直接 make install ，在 `configure` 的时候，在 `ffbuild/config.log` 中会报 C2054 等等错误，如图：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-16.png)

这个C2054 错误非常隐蔽，这个错误不会导致 `configure` 失败，所以看`configure`的输出是成功的，只是执行 `ffmpeg.exe -hwaccels` 显示硬件加速的时候，没有cuda这个选项输出。因此也无法使用硬件编解码。

现在需要把  **ffnvcodec.pc** 放到 `\usr\local\lib\pkgconfig` 目录，如下：

![nvidia-2-1](nvidia\nvidia-2-1.png)

还需要执行以下命令，把 `/usr/local/lib/pkgconfig/` 加入搜索路径：

```
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/:$PKG_CONFIG_PATH
```

------

本文**不需要**安装  [Video_Codec_SDK](https://developer.nvidia.com/nvidia-video-codec-sdk)，那应该是旧版本 CUDA Toolkit 的做法。

我们可以下载一个来看看，Video_Codec_SDK 里面主要是一些 lib 库跟 .h头文件，如下图。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-18.png)

我用 [Everything](https://www.voidtools.com/zh-cn/) 软件 搜索 **nvcuvid.lib** 这个库，这应该是一个导入库，不是静态库。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-19.png)

我电脑是装了 4个版本的 CUDA toolkits，8.0，9.0，9.2，10.1，可以看到 10.1 版本的 CUDA toolkits 并没有出现在 everything 的搜索里面。

也就是说 CUDA toolkits 从 10.1 版本开始就去掉了 nvcuvid.lib 这个库，之前的硬件解码器好像是叫 cuvid，现在好像改了个名字，叫nvdec。

所以本文编译 ffmpeg NVENC NVDEV，不需要下载安装 Video_Codec_SDK，那是旧版的方式。咱们用的 CUDA Toolkits 版本是 10.1，是新版的。

------

到这里， CUDA Toolkit 10.1，nv-codec-headers 9.1.23.3 都已经安装完毕，下面就可以开始用 msys2 + msvc 来编译 FFmpeg 了。

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-4.4-nv-10.1 \
--enable-gpl \
--enable-nonfree \
--enable-cuda-nvcc \
--enable-libnpp \
--enable-shared \
--toolchain=msvc \
--extra-cflags="-I/usr/local/include/nv_sdk_10.1" \
--extra-ldflags="-LIBPATH:/usr/local/lib/x64"
```

扩展知识：msvc 的 link.exe 链接器接受的搜索路径参数是 **-LIBPATH**，gcc 链接器接受的参数是 **-L** ，不要混淆。

从 configure 的输出日志，可以看到以下内容，就代表 cuda 成功了。

![nvidia-2-1-2](nvidia\nvidia-2-1-2.png)

------

FFmpeg-n4.4.1 版本源码在 编译 CUDA 硬件编解码 还会报一个错误，错误提示如下：

```
"libavfilter/vf_scale_cuda_bicubic.ptx.c(1925): fatal error C1091: compiler limit: string exceeds 65535 bytes in length"
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-21.png)

这是一个 FFmpeg-n4.4.1 的一个 [bug](https://trac.ffmpeg.org/ticket/9019)，可以用 此 [patch](https://trac.ffmpeg.org/attachment/ticket/9019/0001-Patch-for-ticket-9019-CUDA-Compile-Broken-Using-MSVC.patch) 来修复。**修复之后要重新 configure**。

------

编译完成之后，执行 **./ffmpeg.exe -hwaccels** 显示 ffmpeg 的硬件加速方法，如下图所示，可以看到 cuda 的选项。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-24.png)

至此，ffmpeg.exe 已经编译完毕。下面找一个高清电影，来测试ffmpeg 的NVENC 硬件编码。

```
./ffmpeg.exe -hwaccel cuvid -i juren.mp4 -vcodec h264_nvenc -acodec copy juren_h264_nvenc.mp4
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-25.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/cuda-0-26.png)

从上图可以看到， cpu的负载很小，而 GPU 直接 99% 满功率了。ffmpeg.exe 硬件编码测试通过。

现在这个 ffmpeg.exe 其实依赖4个dll，`nppc64_10.dll`，`nppicc64_10.dll`，`nppidei64_10.dll`，`nppig64_10.dll`。如下：

![nvidia-2-4](nvidia\nvidia-2-4.png)

所以如果要发布程序，需要把这 4个 dll 跟 ffmpeg.exe 一起发布，如下：

![nvidia-2-5](nvidia\nvidia-2-5.png)



------

**常见错误：**

**1， LINK : fatal error LNK1104: cannot open file 'LIBCMTD.lib'。**

这个错误是因为没用 x64 Native Tools Command Prompt for VS 2019.exe 开启CMD，msys2 没继承vs2019的环境变量，所以找不到 vs2019 目录的 LIBCMTD.lib。

**2，libavfilter/vf_scale_cuda_bicubic.ptx.c(1925): fatal error C1091: compiler limit: string exceeds 65535 bytes in length**

这是一个 FFmpeg-n4.4.1 的一个 [bug](https://trac.ffmpeg.org/ticket/9019)，可以用 此 [patch](https://trac.ffmpeg.org/attachment/ticket/9019/0001-Patch-for-ticket-9019-CUDA-Compile-Broken-Using-MSVC.patch) 来修复。

------

相关阅读：

1，[《NVIDIA官方FFmpeg编译教程》](https://docs.nvidia.com/video-technologies/video-codec-sdk/ffmpeg-with-nvidia-gpu/#compiling-ffmpeg)

2，[《cuda-installation-guide-microsoft-windows》](https://docs.nvidia.com/cuda/cuda-installation-guide-microsoft-windows/index.html#build-customizations-for-existing-projects)

