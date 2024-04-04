# AVString字符串函数库详解—FFmpeg API教程

<div id="meta-description---">FFmpeg 是一个 C语言的项目，C语言标准库的功能比较简陋，所以很多东西都要自己写，造轮子。FFmpeg 已经造了不少轮子，AVString 就是其中一个轮子，用 AVString 函数库，可以使字符串处理更加方便。</div>

FFmpeg 是一个 C语言的项目，C语言标准库的功能比较简陋，所以很多东西都要自己写，造轮子。`FFmpeg` 已经造了不少轮子，`AVString` 就是其中一个轮子，用 `AVString` 函数库，可以使字符串处理更加方便。

`AVString` 函数库的代码在 `libavutil/avstring.h` 里面，本文选取一些常用的函数来讲解。

**1，**`av_strstart()`，检查 `str` 字符串是不是以 `pfx` 开头的，如果是，返回 非 0，同时把 `ptr` 指向 `pfx` 后面的字符。

```
int av_strstart(const char *str, const char *pfx, const char **ptr);
```

**2，**`av_stristart()`，跟 `av_strstart()` 一样，只是不区分大小写。

```
int av_stristart(const char *str, const char *pfx, const char **ptr);
```

上面这两个检查字符串开头的函数确实挺好用，C标准库好像是没有类似的函数。

------

**3，**`av_strnstr()`，跟C标准库函数 `strstr()` 一样，只是限制了搜索长度。

```
char *av_strnstr(const char *haystack, const char *needle, size_t hay_length);
```

**4，**`av_strlcpy()`，根据长度拷贝字符，跟 BSD 的 `strlcpy()` 函数一样。

```
size_t av_strlcpy(char *dst, const char *src, size_t size);
```

**5，**`av_strlcat()`，拼接字符串函数，跟  BSD的 `strlcat()` 类似，但有些许区别。

```
size_t av_strlcat(char *dst, const char *src, size_t size);
```

**6，`av_strlcatf()`，动态参数拼接字符串，感觉这个函数超有用。**

```
size_t av_strlcatf(char *dst, size_t size, const char *fmt, ...) av_printf_format(3, 4);
```

**7，**`av_strnlen()`，获取字符串的长度，但是加了长度限制，估计是为了防止传错指针，字符串没有以 0 结尾，导致死循环。加了长度限制就不会死循环。

```
static inline size_t av_strnlen(const char *s, size_t len)
```

**8，**`av_asprintf()`，类似 GNU 的 `asprintf()` ，返回值是堆指针，需要自己释放内存。

```
char *av_asprintf(const char *fmt, ...) av_printf_format(1, 2);
```

**9，**`av_get_token()`，不知道干嘛的，后面补充。参考：https://blog.csdn.net/sidemap/article/details/123559521

```
char *av_get_token(const char **buf, const char *term);
```

**10，**`av_strtok()`，字符串分割函数，可以把 `hello-jason-please` 字符串按照 `-` 符号分割成 `hello` `jason`  `please`。

```
char *av_strtok(char *s, const char *delim, char **saveptr);
```

**11，**`av_isdigit()`，判断单个字符是不是数字字符，通过把 `char` 转成 `int` 来比较的。

```
/**
 * Locale-independent conversion of ASCII isdigit.
 */
 static inline av_const int av_isdigit(int c){
    return c >= '0' && c <= '9';
}
```

**12，**`av_isgraph()`，判断单个字符是不是图像字符。

```
static inline av_const int av_isgraph(int c){
    return c > 32 && c < 127;
}
```

**13，**`av_isspace()`，判断单个字符是不是空格或者换行符之类的。

```
static inline av_const int av_isspace(int c)
{
    return c == ' ' || c == '\f' || c == '\n' || c == '\r' || c == '\t' ||
           c == '\v';
}
```

**14，**`av_toupper()`，`av_tolower()`，转换大小写。

**15，**`av_isxdigit()`，判断一个字符是不是 16 进制的数字字符。

```
static inline av_const int av_isxdigit(int c)
{
    c = av_tolower(c);
    return av_isdigit(c) || (c >= 'a' && c <= 'f');
}
```

**16，**`av_strcasecmp()`，比较两个字符串是否一样，不区分大小写。

```
int av_strcasecmp(const char *a, const char *b);
int av_strncasecmp(const char *a, const char *b, size_t n);
```

**17，**`av_strireplace()`，后面补充。

**18，**`av_basename()`，后面补充。线程安全的。

**19，**`av_dirname()`，后面补充。线程安全的。

**20，**`av_match_name()`，后面补充。

**21，**`av_append_path_component()`，后面补充。

**22，**`av_escape()`，后面补充。

**23，**`av_utf8_decode()`，后面补充。

**24，**`av_match_list()`，后面补充。

**25，**`av_sscanf()`，后面补充。

---

还有一个我自己最常用的字符串函数，`av_strdup()`，但是这个函数是在 `libavutil/mem.h` 里面的，定义如下：

```
char *av_strdup(const char *s) av_malloc_attrib;
```

这个函数可以很方便的创建一个字符串变量，这个变量是在堆上的，需要自己释放内存。下面对比一下原生的方式跟 用 `av_strdup` 的方式。

```
char* name;
name = malloc(100);
strcpy(name,"loken-ffmpeg");
free(name);
--------------------
name = av_strdup("loken-ffmpeg");
av_freep(&name);
```

可以看到，`av_strdup` 非常地简洁。

------

上面这些函数的使用示例，可以在 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/avstring) 下载，截图如下：

![1-1](avstring\1-1.png)





