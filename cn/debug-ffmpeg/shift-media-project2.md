# ShiftMediaProject具体使用—FFmpeg调试环境搭建

<div id="meta-description---">ShiftMediaProject具体使用</div>

前文《[ShiftMediaProject项目介绍](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/shift-media-project.html)》已经介绍了 项目的 bat 脚本以及依赖逻辑。了解了 bat 脚本的逻辑，我们就不需要使用这个脚本了，可以手动下载特定版本的压缩包。

现在的目录结构如下；

![smp2-1-1](smp2\smp2-1-1.png)

而且 FFmpeg-4.4.r100605 的 project_get_dependencies.bat 脚本的依赖如下：

![smp2-1-2](smp2\smp2-1-2.png)

要编译 FFmpeg ，就需要先编译 以下这些库。

1. gnutls ，版本：3.7.5，有 git 子模块，在百度网盘。
2. bzip2 ，版本：[2-1.0.8](https://github.com/ShiftMediaProject/bzip2/releases/tag/bzip2-1.0.8)
3. mfx_dispatch ，版本：[1.35.r88](https://github.com/ShiftMediaProject/mfx_dispatch/releases/tag/1.35.r88)
4. libiconv，版本：[v1.16-1](https://github.com/ShiftMediaProject/libiconv/releases/tag/v1.16-1)
5. liblzma，版本：[v5.2.5](https://github.com/ShiftMediaProject/liblzma/releases/tag/v5.2.5)
6. libxml2，版本 ：[v2.9.11](https://github.com/ShiftMediaProject/libxml2/releases/tag/v2.9.11)
7. soxr，版本：[0.1.3-1](https://github.com/ShiftMediaProject/soxr/releases/tag/0.1.3-1) 
8. game-music-emu，版本：[0.6.3](https://github.com/ShiftMediaProject/game-music-emu/releases/tag/0.6.3) 
9. modplug，版本：[0.8.9.0.r283](https://github.com/ShiftMediaProject/modplug/releases/tag/0.8.9.0.r283)
10. freetype2，版本：[VER-2-11-0-1](https://github.com/ShiftMediaProject/freetype2/releases/tag/VER-2-11-0-1)
11. fontconfig，版本：[2.13.1-6](https://github.com/ShiftMediaProject/fontconfig/releases/tag/2.13.1-6)
12. libbluray，版本：[1.3.0-1](https://github.com/ShiftMediaProject/libbluray/releases/tag/1.3.0-1)
13. libgpg-error，版本 ：[libgpg-error-1.43](https://github.com/ShiftMediaProject/libgpg-error/releases/tag/libgpg-error-1.43)
14. libgcrypt，版本 ：[libgcrypt-1.9.4](https://github.com/ShiftMediaProject/libgcrypt/releases/tag/libgcrypt-1.9.4)
15. libssh，版本：[libssh-0.9.6](https://github.com/ShiftMediaProject/libssh/releases/tag/libssh-0.9.6)
16. fribidi，版本：[v1.0.11](https://github.com/ShiftMediaProject/fribidi/releases/tag/v1.0.11)
17. harfbuzz，版本：[3.2.0](https://github.com/ShiftMediaProject/harfbuzz/releases/tag/3.2.0)
18. libass，版本：[0.15.2-1](https://github.com/ShiftMediaProject/libass/releases/tag/0.15.2-1)
19. libcdio，版本 ：[release-2.1.0-1](https://github.com/ShiftMediaProject/libcdio/releases/tag/release-2.1.0-1) 
20. libcdio-paranoia，版本：[release-10.2+2.0.1](https://github.com/ShiftMediaProject/libcdio-paranoia/releases/tag/release-10.2%2B2.0.1)
21. SDL，版本：[release-2.0.16](https://github.com/ShiftMediaProject/SDL/releases/tag/release-2.0.16)
22. libilbc，版本：v3.0.3 ，有 git 子模块，在百度网盘。
23. lame，版本：[RELEASE__3_100-1](https://github.com/ShiftMediaProject/lame/releases/tag/RELEASE__3_100-1) 
24. opus，版本：[v1.3.1-1](https://github.com/ShiftMediaProject/opus/releases/tag/v1.3.1-1)
25. libvpx，版本：[v1.11.0](https://github.com/ShiftMediaProject/libvpx/releases/tag/v1.11.0) 
26. x264，版本：[0.164.r3075-1](https://github.com/ShiftMediaProject/x264/releases/tag/0.164.r3075-1)
27. x265，版本：[3.4](https://github.com/ShiftMediaProject/x265/releases/tag/3.4) 
28. xvid，版本：[release-1_3_7](https://github.com/ShiftMediaProject/xvid/releases/tag/release-1_3_7) 
29. speex，版本：1.2.0-4
30. ogg，版本：[v1.3.5](https://github.com/ShiftMediaProject/ogg/releases/tag/v1.3.5) 
31. vorbis，版本：[v1.3.7-1](https://github.com/ShiftMediaProject/vorbis/releases/tag/v1.3.7-1) 
32. theora，版本：[1.2.0alpha1+svn.r135-2](https://github.com/ShiftMediaProject/theora/releases/tag/1.2.0alpha1%2Bsvn.r135-2)

上面的库要按顺序编译的。为了方便读者，上面的压缩包可以在[百度网盘](https://pan.baidu.com/s/1tQWNrjZl-dLDWX0NhXIcuw )一次性下载，提取码：kkj2 

------

上面的项目都是在 Shift Media Project 能搜到的，是子项目，但是还有4个依赖是要在其他地方下载，第一个是 ffnvcodec 。

ffnvcodec  [Version 11.1.5.1](https://github.com/FFmpeg/nv-codec-headers/releases/tag/n11.1.5.1) ，下载之后，把 nv-codec-headers 里面的 ffnvcodec 文件夹，复制到 `D:\shift-media-project\msvc\include`，如下：

![smp2-1-2-1](smp2\smp2-1-2-1.png)

这个其实是 NVIDIA 的硬件编解码器的头文件，不需要 硬件解码也要下载一下，要不 编译 avutil 库就会失败。Shift Media Project 的编译脚本不太智能。

第二个是 opengl 的两个头文件 [glext.h](https://www.khronos.org/registry/OpenGL/api/GL/glext.h) 跟 [wglext.h](https://www.khronos.org/registry/OpenGL/api/GL/wglext.h)，下载之后放到 `D:\shift-media-project\msvc\include\gl\` 目录下。

第三个是 [khrplatform.h](https://www.khronos.org/registry/EGL/api/KHR/khrplatform.h) 头文件，下载之后放到 `D:\shift-media-project\msvc\include\KHR\` 目录下。

第四个是 AMF 的 SDK 头文件。AMF 全称 Advanced Media Framework，在 github 上下载 [AMF Release 1.4.23](https://github.com/GPUOpen-LibrariesAndSDKs/AMF/releases/tag/v1.4.23) ，然后把 `amf/public/include` 目录下的内容拷贝到 `msvc\include\AMF\` 目录，如下：





------

回到 FFmpeg 的依赖项目，开始安装。

gnutls 这个库是比较难安装，因为他里面有很多的 git 子模块，所以不能在 github 直接下 zip 压缩包，你只能 git clone 把目录拷贝下来，然后执行 project_get_dependencies.bat 脚本。但是 gnutls 项目的整个 git 过程会非常慢，这时候你需要一个很好的魔法工具。

小飞机这种魔法工具的代理是有验证的，所以需要把验证加进去才可以代理成功，这个验证码可以在 Windows设置 → 网络和Intelnet  →  代理里面看到，如下：

![smp2-1-3-1](smp2\smp2-1-3-1.png)

然后设置 git 代理即可

```
git config --global http.proxy http://127.0.0.1:19899/pac?auth=xxx\&t=xxx
git config --global https.proxy http://127.0.0.1:19899/pac?auth=xxx\&t=xxx
```

![smp2-1-3](smp2\smp2-1-3.png)

下载解压 完 gnutls 之后，目录如下：

![smp2-1-4](smp2\smp2-1-4.png)

由于 gnutls 依赖 zlib，gmp，nettle。而 nettle 依赖 gmp ，所以需要按顺序编译，先编译 zlib，再编译 gmp，再编译 nettle，最后编译 gnutls 

直接进入 zlib 的 smp 目录，就能看到 vs2019 的解决方案，打开 build 一下就行。如下：

![smp2-1-5](smp2\smp2-1-5.png)

![smp2-1-6](smp2\smp2-1-6.png)

其他的 3个项目也有 smp 文件夹，也是按相同的方法编译，然后在 msvc 输出目录就会看到相关的库，如下：

**提醒：**编译 nettle 的时候里面有个 libhogweed 项目也要编译。

![smp2-1-7](smp2\smp2-1-7.png)

------

现在我们手动下一下 bzip2 的源码，选择版本 2-1.0.8，如下：

提示：最好用 搜索框搜索 Shift Media Project 里面的子项目，因为他有分页，第一页可能没有 bzip2 。

![smp2-1-8](smp2\smp2-1-8.png)

![smp2-1-9](smp2\smp2-1-9.png)

从上图可以看出，实际上有 编译好的 lib 跟 dll 库可以直接用，你也可以下载这些现成的，复制到 `D:\shift-media-project\msvc` 目录就行。

本文为了演示，还是下载 Source code 的压缩包，放到 source 目录下，如下：

![smp2-2-1](smp2\smp2-2-1.png)

然后 用 vs2019 打开 `bzip2-bzip2-1.0.8\SMP\libbz2.sln` 解决方案文件，直接 build，就会在 msvc 目录生成 libbz2d.lib 静态库，如下：

![smp2-2-2](smp2\smp2-2-2.png)

实际上源码编译，跟你直接下他 github 的现成包是一样的。

后面的依赖库就不演示了，操作都是一样的，点击进去 SMP，然后 用 vs2019 编译一下就行。

source 目录下的截图如下：

![smp2-2-2-1](smp2\smp2-2-2-1.png)

**重点提醒：如果你编译上面任何一个项目报错，肯定是依赖项目没有先编译，这时候需要看 project_get_dependencies.bat 脚本里面的内容来判断这个项目到底依赖哪些项目。**

------

编译完这些项目，我已经累死了，所以还是建议直接用他 github 上面编译好的现成包。其实效果是一样的。

------

上面这些 FFmpeg 依赖项目顺利编译完之后，就可以开始编译 FFmpeg 了，一样是打开 SMP 目录的 sln 文件。

打开之后如下：

![smp2-2-3](smp2\smp2-2-3.png)

这样打开项目之后，会发现有一部分项目会加载失败（Load Failed），如下：

![smp2-2-4](smp2\smp2-2-4.png)

**这是因为 我们没有 配置 nasm 跟 yasm ，这两个东西主要是给汇编用了，FFmpeg 里面有一部分汇编代码。**

------

先关闭 vs2019 ，然后下载 [VSNASM.zip](https://github.com/ShiftMediaProject/VSNASM/releases/tag/0.7)，将这个压缩包解压到 `D:\shift-media-project\msvc` 目录，如下：

![smp2-2-5](smp2\smp2-2-5.png)

然后找到 **Developer Command Prompt for VS 2019** 命令行，运行，如下：

![smp2-2-6](smp2\smp2-2-6.png)

执行以下命令：

```
d:
cd D:\shift-media-project\msvc\VSNASM
.\install_script.bat
```

![smp2-2-7](smp2\smp2-2-7.png)

从上图可以看出来，这个脚本安装报错了，我们打开这个 bat 脚本看一下内容，如下：

![smp2-2-7](smp2\smp2-2-8.png)

上图中，REM 开头的是注释，然后 我加了两句代码 echo 了两个 变量 `%SCRIPTDIR%` 跟 `%VCTargetsPath%`，如下：

![smp2-2-7](smp2\smp2-2-9.png)

综上，这个 bat 脚本就是在 拷贝 `nasm.*` 文件的时候出现问题。`%ERRORLEVEL%` 这个变量是 bat 脚本的内置变量，如果命令执行失败，这个 ERRORLEVEL 就不等于 0 。

我们手动执行一下这条 copy 命令，如下：

```
copy /B /Y  .\nasm.*  "C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\MSBuild\Microsoft\VC\v160\BuildCustomizations"
```

![smp2-3-1](smp2\smp2-3-1.png)

可以看到，错误日志是拒绝访问，因为 这是  C 盘的核心目录，所以需要用管理员权限打开  **Developer Command Prompt for VS 2019** 命令行。

![smp2-3-1](smp2\smp2-3-2.png)

用管理员运行之后，再执行一下以上的操作，就不会出问题了。

这样，vs2019 就安装上了 NASM 工具。

------

安装 VSYASM 也是类似的套路，下载 [VSYASM.zip](https://github.com/ShiftMediaProject/VSYASM/releases/tag/0.7)，将这个压缩包解压到 `D:\shift-media-project\msvc` 目录。

然后 用管理员权限打开  **Developer Command Prompt for VS 2019** 命令行。执行以下命令：

```
d:
cd D:\shift-media-project\msvc\VSYASM
.\install_script.bat
```

![smp2-3-3](smp2\smp2-3-3.png)

------

至此，yasm 跟 nasm 都安装进去 vs2019 了，重新启动 vs2019，然后打开之前的 FFmpeg-4.4.r100605 项目。如下：

![smp2-3-3](smp2\smp2-3-4.png)

可以看到，这些库都没有报 加载失败（Load Failed）了。

------

然后按下面顺序编译 FFmpeg 的各个库 跟 ffmpeg.exe 即可。

1. libswscale
2. libswresample
3. libpostproc
4. libavutil
5. libavformat
6. libavfilter
7. libavdevice
8. libavcodec
9. ffmpeg.exe

**重点提示：FFmpeg 项目会有缓存，如果找不到头文件，可以尝试关闭重新打开。**

------

上面的项目编译成功之后，就会生成一个 接近 100M的 ffmpeg.exe ，如下：

![smp2-3-6](smp2\smp2-3-6.png)

这个移植的 FFmpeg 项目，实际上把所有 的编码器都集成在里面，如果想自定义一些库，可以使用这个项目 [FFVS-Project-Generator](https://github.com/ShiftMediaProject/FFVS-Project-Generator)

------

扩展知识：各个项目默认是编译出静态库的，然后你想用动态库，可以换下 Configuration，他都配置好给你的了。如下：

![smp2-3-6-1](smp2\smp2-3-6-1.png)

从上图可以看出，调试版本，正式版本，静态库版本，动态库版本都有。

------

下面演示一下 断点调试 ffmpeg.c 的情况，先把 ffmpeg Program 设置为 Starup Project，如下：

![smp2-3-7](smp2\smp2-3-7.png)

然后菜单栏的 绿色三角调试按钮就可以点击了，顺便在 ffmpeg.c 的 main 函数那里打个断点。

项目的 debug 配置好像有问题，如果调试的时候找不到 ffmpegd.exe ，可以按我下图这样修改一下：

![smp2-3-8](smp2\smp2-3-8.png)

调试界面如下：

![smp2-3-9](smp2\smp2-3-9.png)

