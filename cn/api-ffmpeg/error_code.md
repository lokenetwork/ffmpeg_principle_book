# FFmpeg的错误码—FFmpeg API教程

<div id="meta-description---">FFmpeg的错误码</div>

大部分开源项目都会封装一下错误码，FFmpeg 也不例外。FFmpeg 对 错误码 以及 相关的 API 函数 的定义是在 `libavutil/error.h` 里面的，如下：

```
#define FFERRTAG(a, b, c, d) (-(int)MKTAG(a, b, c, d))

#define AVERROR_BSF_NOT_FOUND      FFERRTAG(0xF8,'B','S','F') ///< Bitstream filter not found
#define AVERROR_BUG                FFERRTAG( 'B','U','G','!') ///< Internal bug, also see AVERROR_BUG2
#define AVERROR_BUFFER_TOO_SMALL   FFERRTAG( 'B','U','F','S') ///< Buffer too small
#define AVERROR_DECODER_NOT_FOUND  FFERRTAG(0xF8,'D','E','C') ///< Decoder not found
#define AVERROR_DEMUXER_NOT_FOUND  FFERRTAG(0xF8,'D','E','M') ///< Demuxer not found
#define AVERROR_ENCODER_NOT_FOUND  FFERRTAG(0xF8,'E','N','C') ///< Encoder not found
#define AVERROR_EOF                FFERRTAG( 'E','O','F',' ') ///< End of file
#define AVERROR_EXIT               FFERRTAG( 'E','X','I','T') ///< Immediate exit was requested; the called function should not be restarted
#define AVERROR_EXTERNAL           FFERRTAG( 'E','X','T',' ') ///< Generic error in an external library
#define AVERROR_FILTER_NOT_FOUND   FFERRTAG(0xF8,'F','I','L') ///< Filter not found
#define AVERROR_INVALIDDATA        FFERRTAG( 'I','N','D','A') ///< Invalid data found when processing input
#define AVERROR_MUXER_NOT_FOUND    FFERRTAG(0xF8,'M','U','X') ///< Muxer not found
#define AVERROR_OPTION_NOT_FOUND   FFERRTAG(0xF8,'O','P','T') ///< Option not found
#define AVERROR_PATCHWELCOME       FFERRTAG( 'P','A','W','E') ///< Not yet implemented in FFmpeg, patches welcome
#define AVERROR_PROTOCOL_NOT_FOUND FFERRTAG(0xF8,'P','R','O') ///< Protocol not found
....省略....
```

`MKTAG()` 宏函数的定义在  `libavutil/common.h` 里面，如下：

```
#define MKTAG(a,b,c,d) ((a) | ((b) << 8) | ((c) << 16) | ((unsigned)(d) << 24))
```

**因此，FFmpeg 封装错误码的方法，实际上就是把 英文字符对应的 ASCII 码拼接起来，转成数字拼接起来。**

**由于一个 英文字符 占  8 位（1个字节），所以 4 个字符，刚刚好可以放进去 一个 `int` 的内存里面，一个 `int` 是 4 个字节大小。**

------

下面通过一个代码实例来演示一下错误码，下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/error_code)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

```
#include <stdio.h>
#include "libavutil/avutil.h"

int main()
{
    printf("AVERROR_BUG  is 0x%X \n",AVERROR_BUG);
    printf("AVERROR_EXIT is 0x%X \n",AVERROR_EXIT);
    printf("AVERROR_EOF  is 0x%X \n",AVERROR_EOF);
    return 0;
}
```

运行结果如下：

![1-1](error_code\1-1.png)

大家可以网上搜一下 ，字符 B 的 ASCII 码是不是 `0xDE`，字符 U 的 ASCII 码是不是 `0xB8`。

------

有时候我们调一些 FFmpeg 的函数的时候会发生错误，例如调 `avcodec_receive_packet()` 的时候，编码器内部报错了。这时候如果想需要显示具体的错误给用户看，就可以用到下面两个函数来**把 错误码 转成字符串**。

**1，**`av_err2str()`，这是一个比较方便的宏函数，定义如下：

```
/**
 * Convenience macro, the return value should be used only directly in
 * function arguments but never stand-alone.
 */
#define av_err2str(errnum) \
    av_make_error_string((char[AV_ERROR_MAX_STRING_SIZE]){0}, AV_ERROR_MAX_STRING_SIZE, errnum)
```

`av_err2str()` 不是返回的堆指针，而是栈指针，所以你不用释放内存，他这种传参是初始化了一个栈变量，然后把栈变量指针传进去的。

这种写法，跟你在调函数之前，创建一个栈变量，然后把栈变量指针丢进去函数是一样的，**他这种写法很值得学习**。

不过这种写法在 C++ 是不支持的，如果在 C++ 项目调用了 `av_err2str()` 宏函数，会编译报错。这时候可以使用下面的 `av_strerror()` 函数。

**2，**`av_strerror()`，这个函数有点麻烦，你需要先定义个栈内存，或者堆内存，然后传给它。定义如下：

```
int av_strerror(int errnum, char *errbuf, size_t errbuf_size);
```

------

上面两个 API 函数的示例代码如下：

```
printf("AVERROR_EOF  is %s \n",av_err2str(AVERROR_EOF));

char str[AV_ERROR_MAX_STRING_SIZE] = {0};
av_strerror(AVERROR_BUG,str,AV_ERROR_MAX_STRING_SIZE);
printf("AVERROR_EOF  is %s \n",str);
```

运行结果如下：

![1-4](error_code\1-4.png)

FFmpeg 的错误码介绍完毕。

