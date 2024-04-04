# configure变量分析-终章—FFFmpeg configure源码分析

**1，THREADS_LIST**，线程列表变量，这个变量需要详解讲解一些。

```
THREADS_LIST="
    pthreads
    os2threads
    w32threads
"
```

里面有 3 个值，这个 3 个值是什么呢？

第一，pthread 是 Linux 系统的 线程 api。

第二，os2threads 是 `OS/2` 系统的 线程 api，`OS/2` 是 IBM 开发的系统，现在很少见了。

第三，w32threads，w32threads 是 Windows 系统的 线程 api，创建线程的函数 是 `CreateThread`

------

`THREADS_LIST` 这个变量主要在 `AUTODETECT_LIBS` 被使用。因为 autodetect 的库，默认都会设为 yes，然后如果检测不通过就会设置为 no。

所以上面 3 个值，肯定有一个会是 yes。其他两个是 no。

FFmpeg 的编译默认就会开启多线程编解码。



------

**2，ATOMICS_LIST**，原子数据列表，不同操作系统的原子数据结构都不太一样。这个变量主要被线程依赖，如下：

![configure-var2-1-1](configure-var2\configure-var2-1-1.png)

------

**3，AUTODETECT_LIBS**，自动检测的库，默认设为 yes，检测不通过再设置为 no

------

**4，ARCH_LIST** ，指令集架构列表。这里注意，除非开启了 asm 汇编优化才会用到 这个 `ARCH_LIST` 里面变量。

如果 configure 的时候关闭了 asm ，arch 变量就会变成 c ，如下，C 语言是跟指令架构无法的。

```
enabled asm || { arch=c; disable $ARCH_LIST $ARCH_EXT_LIST; }
```

------

**5，ARCH_EXT_LIST_ARM**，ARM 指令集里面的微架构，ARM 指令集里面有很多子分支的。

后面的 **ARCH_EXT_LIST_MIPS**，**ARCH_EXT_LIST_LOONGSON**，**ARCH_EXT_LIST_X86**  变量都是特定指令架构里面的微架构。

------

**6，ARCH_EXT_LIST** ，所有微架构指令集里面的集合。

------

**7，ARCH_FEATURES**， 指令集的特性，主要被 check_deps 函数 使用。

------

**8，BUILTIN_LIST，HAVE_LIST_PUB，HAVE_LIST_CMDLINE**，这些变量主要都是被 check_deps 函数 使用。check_deps 检测通过就会设为 yes。

------

**9，HEADERS_LIST**，头文件列表，主要被 check_deps 函数 使用。

------

**10，INTRINSICS_LIST**，暂时不知道这个变量干什么的。

------

**11，COMPLEX_FUNCS，MATH_FUNCS，SYSTEM_FEATURES，SYSTEM_FUNCS**，这些变量都会被被 check_deps 函数 使用。check_deps  会怎么检测这些变量呢？

说实话，这些函数大部分没有 deps，没有依赖，没有 suggest，所以 check_deps 什么都不会干。所以我也不太明白定义这些变量干什么？

里面的 getaddrinfo 之类的函数的检测是通过 另外的地方检测，所以这些 `SYSTEM_FUNCS` 变量的作用就是转存进去 `config.h` 里面。

------

**12，HAVE_LIST**，一个大杂烩，会被  check_deps 函数 使用。也会转存进去  `config.h` 里面。

------

**13，CONFIG_EXTRA**，不能在命令行使用的一些配置项。会被  check_deps 函数 使用。也会转存进去  `config.h` 里面。

------

**14，CMDLINE_SELECT** ，可以在命令行指定通过 `--enable-?*|--disable-?* ` 使用的的一些选项集合

![configure-var2-1-1-2](configure-var2\configure-var2-1-1-2.png)

------

**15，PATHS_LIST**，路径列表。会被 `set_default` 函数使用，设置默认的路径。

------

**16，CMDLINE_SET**，可以在命令行通过指定的选项集合，例如可以设置 `cc` 来指定编译器，设置 `ld` 来指定链接器，

![configure-var2-1-2](configure-var2\configure-var2-1-2.png)

------

**17，CMDLINE_APPEND**，可以附加的命令行选项，不会替换调之前的内容，只会附加上去，定义如下：

```
CMDLINE_APPEND="
    extra_cflags
    extra_cxxflags
    extra_objcflags
    host_cppflags
"
```

`extra_cflags` 这个命令行参数是编译 FFmpeg 的时候经常用的，可以传递参数给 编译器。还有一个 `extra_ldflags` 没在里面，我个人觉得不够统一，

![configure-var2-1-3](configure-var2\configure-var2-1-3.png)

从上图可以看出，链接器的选项，在另一个地方。

------

至此，configure 脚本里面大部分的通用变量已经讲解完毕，从  2551 行开始，就是一些 shell 的具体逻辑，如下：

```
# code dependency declarations

# architecture extensions

armv5te_deps="arm"
armv6_deps="arm"
armv6t2_deps="arm"
armv8_deps="aarch64"
neon_deps_any="aarch64 arm"
intrinsics_neon_deps="neon"
vfp_deps_any="aarch64 arm"
vfpv3_deps="vfp"
setend_deps="arm"
```

