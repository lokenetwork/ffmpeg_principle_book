# configure变量分析-A章—FFFmpeg configure源码分析

大概从 1642 行开始，configure 脚本就开始定义很多很多的变量，本章内容就来简单介绍一下这些变量的作用。

------

**1，AVCODEC_COMPONENTS，AVCODEC_COMPONENTS_LIST**，定义一些组件，这两个变量的定义如下：

```
AVCODEC_COMPONENTS="
    bsfs
    decoders
    encoders
    hwaccels
    parsers
"
```

```
AVCODEC_COMPONENTS_LIST="
    $BSF_LIST
    $DECODER_LIST
    $ENCODER_LIST
    $HWACCEL_LIST
    $PARSER_LIST
"
```

`bsfs` 全称 是 `BitStreamFilter`，后面的 s 代表复数。

BitStreamFilter 翻译过来就是 位流滤镜，实际上这些滤镜主要是处理格式问题。例如 h264_mp4toannexb ，就是处理 h264 的格式的。



------

**2，AVDEVICE_COMPONENTS**，定义输入输出组件。

------

**3，AVFILTER_COMPONENTS**，定义 滤镜组件。

------

**4，AVFORMAT_COMPONENTS**，定义容器组件，里面有网络协议。

------

**5，COMPONENT_LIST**，以上的组件集合。

------

**6，EXAMPLE_LIST**，example 文件的例子集合。

------

**7，EXTERNAL_AUTODETECT_LIBRARY_LIST**，自动匹配的库列表。这个比较重要，我举个例子详细讲一下是怎么自动匹配的。

`EXTERNAL_AUTODETECT_LIBRARY_LIST` 变量里面有 `sdl2` ，所以我以 `sdl2` 举例。

在 4142 行左右，会用 weak 的方式默认把 自动匹配的库列表设为 yes。

```
#TODO: switch to $AUTODETECT_LIBS when $THREADS_LIST is supported the same way
enable_weak $EXTERNAL_AUTODETECT_LIBRARY_LIST
enable_weak $HWACCEL_AUTODETECT_LIBRARY_LIST
```

然后在 6556 行会 重新检测 sdl2 是否能正常使用，不能使用就会 重新设置为 no。如下：

```

if enabled sdl2; then
    SDL2_CONFIG="${cross_prefix}sdl2-config"
    test_pkg_config sdl2 "sdl2 >= 2.0.1 sdl2 < 2.1.0" SDL_events.h SDL_PollEvent
    if disabled sdl2 && "${SDL2_CONFIG}" --version > /dev/null 2>&1; then
        sdl2_cflags=$("${SDL2_CONFIG}" --cflags)
        sdl2_extralibs=$("${SDL2_CONFIG}" --libs)
        test_cpp_condition SDL.h "(SDL_MAJOR_VERSION<<16 | SDL_MINOR_VERSION<<8 | SDL_PATCHLEVEL) >= 0x020001" $sdl2_cflags &&
        test_cpp_condition SDL.h "(SDL_MAJOR_VERSION<<16 | SDL_MINOR_VERSION<<8 | SDL_PATCHLEVEL) < 0x020100" $sdl2_cflags &&
        check_func_headers SDL_events.h SDL_PollEvent $sdl2_extralibs $sdl2_cflags &&
            enable sdl2
    fi
    if test $target_os = "mingw32"; then
        sdl2_extralibs=$(filter_out '-mwindows' $sdl2_extralibs)
    fi
fi
```

`EXTERNAL_AUTODETECT_LIBRARY_LIST` 变量里面的 `iconv`  也是类似的套路，先设为 yes，然后再检测，检测不通过就设置 no。

下面是 6808 行的代码。

```
# Funny iconv installations are not unusual, so check it after all flags have been set
if enabled libc_iconv; then
    check_func_headers iconv.h iconv
elif enabled iconv; then
    check_func_headers iconv.h iconv || check_lib iconv iconv.h iconv -liconv
fi
```

因此这些 autodetect 的库，并不是一个必须库，少了这些库，ffmpeg.exe 还是可以跑，只是缺少一些功能。

------

**8，EXTERNAL_LIBRARY_GPL_LIST**，这里面是 GPL 协议的库，x264 ，x265 这些都是 GPL 协议。

这个变量在 configure 里面的应用如下：

```
map "die_license_disabled gpl"      $EXTERNAL_LIBRARY_GPL_LIST $EXTERNAL_LIBRARY_GPLV3_LIST
```

就是判断 gpl 是否打开，`--enable-gpl` 可以把 gpl 变量设置为 yes，如果 gpl 没打开，而 x264 这些库又设置为 yes，就是冲突了，会报错退出。

------

**9，EXTERNAL_LIBRARY_NONFREE_LIST**，nofree 

这些是不能免费商用的库。如果 configure 的时候没使用 `--enable-nonfree` 而是单独把这些 库设为 yes，就会报错，检测代码如下：

```
enabled gpl && map "die_license_disabled_gpl nonfree" $EXTERNAL_LIBRARY_NONFREE_LIST
```

------

**10，EXTERNAL_LIBRARY_VERSION3_LIST**， GPL version3 的库，使用这些库要先指定 `--enable-version3` 

------

**11，EXTERNAL_LIBRARY_GPLV3_LIST**，这个变量感觉是 GPL 跟 GPL3 共用的。

------

**12，EXTERNAL_LIBRARY_LIST**，这个变量里面包含了 FFmpeg 使用的所有 非系统的外部库。通常会先设置为 no，如下：

```
# Avoid external, non-system, libraries getting enabled by dependency resolution
disable $EXTERNAL_LIBRARY_LIST $HWACCEL_LIBRARY_LIST
```

------

**13，HWACCEL_AUTODETECT_LIBRARY_LIST**，自动检测的硬件加速相关的库，默认会先设置为 yes，如下：

```
enable_weak $HWACCEL_AUTODETECT_LIBRARY_LIST
```

FFmpeg 的 套路是这样的，一个变量为 yes， 他就会去检测 这个库是否能正常使用，如果不能，就会改为 no。如果这个 库是必须的，那就会报错。如果这个库不是必须的，他就会悄悄改为 no，不报错。

```
enabled amf &&
    check_cpp_condition amf "AMF/core/Version.h" \
        "(AMF_VERSION_MAJOR << 48 | AMF_VERSION_MINOR << 32 | AMF_VERSION_RELEASE << 16 | AMF_VERSION_BUILD_NUM) >= 0x0001000400090000"
```

例如里面 的 amf 加速，就是上面这样检测是否能正常使用的。

------

**14，EXTRALIBS_LIST**，这些是默认需要的库。

```
# catchall list of things that require external libs to link
EXTRALIBS_LIST="
    cpu_init
    cws2fws
"
```

------

**15，HWACCEL_LIBRARY_NONFREE_LIST**，需要收费的硬件加速库。cuda 居然是商用需要收费的

```
HWACCEL_LIBRARY_NONFREE_LIST="
    cuda_nvcc
    cuda_sdk
    libnpp
"
```

------

**17，HWACCEL_LIBRARY_LIST**，硬件加速类库，默认全设置为 no。

configure 的套路就是 自动检测的库就先设置为 yes，设为 yes 就会检测是否正常使用。不需要自动检测的库，也就是要手动开启的库，默认设置为 no。

------

**18，DOCUMENT_LIST**，文档列表，默认全设置未 yes。

------

我提前剧透一下，`configure` 这个脚本搞这么多变量，设为 `yes`，或者 `no`。实际上就是为了 往 编译器，链接器 传递一些参数而已。

即便 `configure` 的 8 千行代码 玩出花来，最后也只是往 编译器，链接器 传递参数而已。

------

**19，FEATURE_LIST**，特性列表，这里面不是一些库，而是一些特性，例如 shared 跟 static，就是生成静态库动态库。

`FEATURE_LIST` 这个变量会丢进去 `CMDLINE_SELECT` 里面，被命令行参数使用。如下：

![configure-var-1-5](configure-var\configure-var-1-5.png)

------

**20，LIBRARY_LIST**，FFmpeg 自己的库列表，有链接顺序的，不能乱。

------

**21，LICENSE_LIST**，版权列表。

```
LICENSE_LIST="
    gpl
    nonfree
    version3
"
```

------

**22，PROGRAM_LIST**，FFmpeg 的可执行文件列表。

```
PROGRAM_LIST="
    ffplay
    ffprobe
    ffmpeg
"
```

------

**22，SUBSYSTEM_LIST**，子系统列表，什么是子系统？实际上这里面不是传统意义上的子系统，而是一些子系统的特性，例如 `fast_unaligned`，这些 `aarch64` ，`ppc` ，`x86` 芯片都有 `fast_unaligned` 这个特性。

`SUBSYSTEM_LIST` 最后 丢进去 `CONFIG_LIST` 然后被 `check_deps` 函数使用。

我个人感觉 `SUBSYSTEM_LIST` 这个变量是为了启用一些 汇编指令优化。

------

**23，CONFIG_LIST**，这是一个大杂烩变量，库跟特性都有。有 3 个地方用到这个变量，如下：

![configure-var-1-6](configure-var\configure-var-1-6.png)

![configure-var-1-6](configure-var\configure-var-1-7.png)

![configure-var-1-6](configure-var\configure-var-1-8.png)

最重要的是第三种图的使用，调了 `print_config` 函数，之前在 [《configure函数分析-C章》](https://ffmpeg.xianwaizhiyin.net/build-ffmpeg/configure-function3.html) 讲过这个函数的使用，就是把 yes ，no 转成 0 跟 1，存进去 .h 头文件 跟 mak 文件。

如下：

![configure-function3-1-1](https://ffmpeg.xianwaizhiyin.net/build-ffmpeg/configure-function3/configure-function3-1-1.png)















