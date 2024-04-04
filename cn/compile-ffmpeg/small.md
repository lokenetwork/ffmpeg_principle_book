# 如何裁剪FFmpeg—FFmpeg编译教程

<div id="meta-description---">如何裁剪FFmpeg</div>

众所周知，`ffmpeg.exe` 命令行的功能非常强大，但是他的体积非常庞大，如果是[静态编译](https://ffmpeg.xianwaizhiyin.net/compile-ffmpeg/static.html)，`ffmpeg.exe` 的大小会高达到 20M，如下：

![1-1](small\1-1.png)

但是，在一些嵌入式设备，或者某些场景下，不需要用到这么多的功能，例如不需要捕捉摄像头的输入，不需要进行网络通信，等等。

这时候，可不可以把这些不需要的功能裁剪掉，让 `ffmpeg.exe` 或者 `dll` 的体积或者依赖更小呢？

答：是可以的，FFmpeg 的开发者早就想到了这个需求，所以在 `configure` 脚本里面定义了很多裁剪的选项，如下：

```
Component options:
  --disable-avdevice       disable libavdevice build
  --disable-avcodec        disable libavcodec build
  --disable-avformat       disable libavformat build
  --disable-swresample     disable libswresample build
  --disable-swscale        disable libswscale build
  --disable-postproc       disable libpostproc build
  --disable-avfilter       disable libavfilter build
  --enable-avresample      enable libavresample build (deprecated) [no]
  --disable-pthreads       disable pthreads [autodetect]
  --disable-w32threads     disable Win32 threads [autodetect]
  --disable-os2threads     disable OS/2 threads [autodetect]
  --disable-network        disable network support [no]
  --disable-dct            disable DCT code
  --disable-dwt            disable DWT code
  --disable-error-resilience disable error resilience code
  --disable-lsp            disable LSP code
  --disable-lzo            disable LZO decoder code
  --disable-mdct           disable MDCT code
  --disable-rdft           disable RDFT code
  --disable-fft            disable FFT code
  --disable-faan           disable floating point AAN (I)DCT code
  --disable-pixelutils     disable pixel utils in libavutil
```

---

下面我们就以 `msvc` 的编译环境来试一下 `--disable-avdevice` ，`--disable-network` 选项的作用，参考《[用msys2与msvc编译FFmpeg](https://ffmpeg.xianwaizhiyin.net/debug-ffmpeg/msys2-msvc.html)》，修改一下 `configure` 的参数，如下：

```
./configure \
--prefix=/home/loken/ffmpeg/build64/ffmepg-n4.4.1-small \
--enable-gpl \
--enable-nonfree \
--disable-shared \
--disable-optimizations \
--disable-asm \
--disable-stripping \
--toolchain=msvc \
--disable-avdevice \
--disable-network
```

编译完成之后，可以看到 `ffmpeg.exe` 的大小只有 19.8M，少了一点，如下：

![1-2](small\1-2.png)

而且 `ffmpeg.exe` 也不再依赖 `gdi32.dll`，等库，`gdi32.dll` 是用来捕捉摄像头之类用的。

![1-3](small\1-3.png)



