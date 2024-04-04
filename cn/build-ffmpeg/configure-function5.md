# configure函数分析-终章—FFFmpeg configure源码分析

configure 里面定义了 几十个 shell 函数，本文就用示例代码来介绍这些函数的具体用法。有些函数只给了函数名，没贴代码是为了节省篇幅。

------

**1，check_func** ，检查 某个函数是否可以正常使用，如果可以使用，把这个函数对应的变量设置为 yes。

![configure-function5-1-1](configure-function5\configure-function5-1-1.png)

------

**2，check_complexfunc**，主要用来 检查 `cabs` 跟 `cexp` 这些复数函数是否能正常使用。能就设置 为 yes。

`complex.h` 是C标准库的一个头文件。

![configure-function5-1-2](configure-function5\configure-function5-1-2.png)



------

**3，check_mathfunc**，检测数学相关的函数，通过即设为 yes。

------

**4，check_func_headers**，这个函数比较重要，需要详解讲解一下他的逻辑。代码如下：

![configure-function5-1-3](configure-function5\configure-function5-1-3.png)

因为这个函数不太方便写小例子演示，我直接讲逻辑。上面图片的逻辑是这样的。

`check_func_headers` 这个函数就是可以引入 `SDL_events.h` 头文件，然后测试 `SDL_PollEvent` 函数是否能正常编译。

可以认为他是同时检测 头文件是否存在，函数能否正常使用。如果检测都通过，头文件变量，函数变量，全都会通过 `enable` 函数设置为 yes。

------

**5，check_class_headers_cpp，**检查类的实例化有无问题，没问题设为 yes ，好像他这个函数写错了，`$funcs` 应该换成 `$classes` ，暂时不管

![configure-function5-1-4](configure-function5\configure-function5-1-4.png)

------

**6，test_cpp_condition**，检测某段宏代码结果是否为真，他没有运行编译之后的可执行文件，里面都是宏代码，他利用宏来判断的，如下：

![configure-function5-1-5](configure-function5\configure-function5-1-5.png)

**宏代码的好处是，可以在编译期间拿到结果，不需要运行。**

------

**7，check_cpp_condition**，`check_cpp_condition` 里面调了 `test_cpp_condition` 来检测某段宏代码，检测通过设置 `$1` 为 yes。

![configure-function5-1-6](configure-function5\configure-function5-1-6.png)

从上图可以看出，如果检测通过，就会把 `$thumb` 变量 设为 yes。

------

**8，test_cflags_cc**，检测一段宏是否为真。

------

**9，check_lib**，检查一个库的使用是否有问题，没问题就设为 yes。

![configure-function5-1-7](configure-function5\configure-function5-1-7.png)

------

**10，check_lib_cpp**，类似 check_lib_cpp

------

**11，test_pkg_config**，这个函数经常使用，比较重要，会重点讲解。

`test_pkg_config` 函数，是测试一下 使用 `pkg-config` 命令引入一些外部库是否有问题。以 SDL2 为例，代码如下：

![configure-function5-1-8](configure-function5\configure-function5-1-8.png)

上面的代码，第一步，他会先用 `pkg-config` 命令查询 SDL2 的版本号，如果版本不符合，直接 return 退出。

然后调 `check_func_headers` 引入 `SDL_events.h` ，还有测试一下 `SDL_PollEvent` 函数能否正常使用。

如果以上检测都通过，就会设置 `$sdl2` 为 yes 。相关的 `$sdl2_cflags` 跟 `$sdl2_extralibs` 也会被设置。

`pkg-config` 命令的用法在[《FFmpeg引入SDL扩展》](https://ffmpeg.xianwaizhiyin.net/compile-ffmpeg/sdl.html)，有详解讲解。

------

**12，check_pkg_config**，`check_pkg_config` 只是对 `test_pkg_config` 做了一点封装，加了一下 `add_cflags`，如下：

![configure-function5-1-9](configure-function5\configure-function5-1-9.png)

`check_pkg_config` 使用 `pkg-config` 的时候没有指定版本，只填了库名，这也是可以的。这是不限制版本。

------

**13，test_exec**，这个函数比较有趣，会尝试运行编译出来的可执行文件，如果是跨平台编译，就不运行。

![configure-function5-2-1](configure-function5\configure-function5-2-1.png)

`test_ld` 这个命令会把生成的文件写进去 $TMPE 变量指定的文件。

------

**14，check_exec_crash**，这个函数主要是 x86 环境下用了检测一下 `ebp` 寄存器的，检测通过会设置 `ebp_available` 为 yes ，如下：



![configure-function5-2-2](configure-function5\configure-function5-2-2.png)

------

**15，check_type**，检测某些结构体是否可以正常编译，如下：

![configure-function5-2-3](configure-function5\configure-function5-2-3.png)

------

**16，check_struct**，检测结构体里面的某个成员是否存在。存在设为 yes

![configure-function5-2-4](configure-function5\configure-function5-2-4.png)

------

**分析到这里，说一个 FFmpeg 开发者 写shell 的习惯，test_xxx 之类的函数，都是只返回 非0 或者 0 ，而 check_xxx 通常会调 test_xxx，根据返回值设置一些变量为 yes。**

------

**17，check_builtin**，编译一段代码，通过之后把变量设为 yes。

------

**18，check_compile_assert**，编译一段宏代码。这个函数其实跟  `check_cpp_condition` 是一样的，应该可以直接用 `check_cpp_condition`，都是编译宏代码。

我个人猜测是 这个 configure 是由不同的人写的，有重复功能的函数。

------

**19，check_cc**，也是编译一段代码，通过之后设置一个变量。

------

**20，require**，检查某些库是否编译没有问题，有问题就报错退出。

![configure-function5-2-5](configure-function5\configure-function5-2-5.png)

------

**21，require_cc** ，调用 `test_cc` 编译一段代码，如果有编译错误，直接退出

------

**22，require_cpp** ，也是编译一段代码，如果有编译错误，直接退出

------

**23，require_headers** ，通过编译来检查头文件存在不存在，不存在报错退出。

------

**24，require_cpp_condition**，也是编译一段代码，如果有编译错误，直接退出

------

**25，require_pkg_config**，调 check_pkg_config 检查一些库有无问题，有问题退出。

------

**分析到这里，说一个 FFmpeg 开发者 写shell 的另一习惯， require_xxx 通常会调 check_xxx，如果 check_xxx 返回 非 0 直接退出。**

------

后面还有几个函数，`test_host_cc`，`test_host_cpp`，`check_host_cppflags`，`check_host_cflags`，`check_host_cpp_condition`，这些函数的套路都是一样的，就没必要讲解了。

------

最后，把 configure 的函数分类一下。

1，test_xxx，执行一段逻辑，会返回 0 或者 非 0。

2，check_xxx，里面会调 text_xxx，根据 返回值设置 某些 变量为 yes。

3，require_xxx，里面会调 check_xxx，如果 check_xxx 返回 非 0 ，立即报错退出。

