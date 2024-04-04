# FFmpeg的日志函数av_log—FFmpeg API教程

<div id="meta-description---">FFmpeg 项目里面 输出日志信息的函数是 av_log，使用 av_log 函数可以在二次开发的时候，比较方便地做一些跟踪跟记录，可以把 av_log 作为 printf 函数的替代品。</div>

FFmpeg 项目里面 输出日志信息的函数是 `av_log`，使用 `av_log` 函数可以在二次开发的时候，比较方便地做一些跟踪跟记录，可以把 `av_log` 作为 `printf` 函数的替代品。

`av_log` 函数的定义如下：

```
/**
 * Send the specified message to the log if the level is less than or equal
 * to the current av_log_level. By default, all logging messages are sent to
 * stderr. This behavior can be altered by setting a different logging callback
 * function.
 * @see av_log_set_callback
 *
 * @param avcl A pointer to an arbitrary struct of which the first field is a
 *        pointer to an AVClass struct or NULL if general log.
 * @param level The importance level of the message expressed using a @ref
 *        lavu_log_constants "Logging Constant".
 * @param fmt The format string (printf-compatible) that specifies how
 *        subsequent arguments are converted to output.
 */
void av_log(void *avcl, int level, const char *fmt, ...) av_printf_format(3, 4);
```

`av_log` 函数的参数如下：

**1，**`void *avcl`，这个指针会强制转成 `AVClass *` ，如果提供了，就会把 `AVClass` 的信息也保存到日志，通常传 NULL 即可。

**2，**`int level`，设置此日志的**级别**，只要比 `av_log_level` 小，`av_log()` 函数就会把这个日志输出到控制台。 `av_log_level` 是一个全局变量。默认是 `AV_LOG_INFO`，

**3，**` const char *fmt` 跟 `...`，这两个参数实际上就跟 `printf()` 函数的用法类似，是可变参数。

------

FFmpeg 的日志一共有 9 个级别，定义在 `libavutil/log.h` 里面，如下：

```
#define AV_LOG_QUIET    -8
#define AV_LOG_PANIC     0
#define AV_LOG_FATAL     8
#define AV_LOG_ERROR    16
#define AV_LOG_WARNING  24
#define AV_LOG_INFO     32
#define AV_LOG_VERBOSE  40
#define AV_LOG_DEBUG    48
#define AV_LOG_TRACE    56
#define AV_LOG_MAX_OFFSET (AV_LOG_TRACE - AV_LOG_QUIET)
```

------

下面就来演示一下，如何使用 `av_log` 这个函数，代码下载地址：[GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/log)，编译环境是 Qt 5.15.2 跟 MSVC2019_64bit 。

![1-3](log\1-3.png)

可以看到，日志默认是输出到 控制台里面的。

------

但是有一些场景，可能需要我们把日志保存到文件，或者保存到数据库，或者通过网络保存到日志服务器里面，这时候怎么做呢？

**答：**可以使用 `av_log_set_callback` 函数来自定义，代码如下：

![1-4](log\1-4.png)

可以看到，我定义了一个回调，来接受参数，然后打印到控制台，可以在这里输出到对应的日志文件，为了偷懒，我没操作文件打开写入。只是简单 `printf` 一下。

扩展知识：`AVBPrint` 跟 `va_list` 是 FFmpeg 自己的结构，源码有注释，对着抄就行。推荐阅读[《深入理解FFmpeg AVBPrint》](https://mp.weixin.qq.com/s/hIQ8Tc2e2cidAGWzgO24rQ)



