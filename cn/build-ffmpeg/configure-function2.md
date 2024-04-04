# configure函数分析-B章—FFFmpeg configure源码分析

configure 里面定义了 几十个 shell 函数，本文就用示例代码来介绍这些函数的具体用法。

------

**1，sanitize_var_name，set_sanitized，get_sanitized** ，这3个是一组函数，用了规范化变量，示例如下：

```
#!/bin/sh
sanitize_var_name(){
    echo $@ | sed 's/[^A-Za-z0-9_]/_/g'
}
set_sanitized(){
    var=$1
    shift
    eval $(sanitize_var_name "$var")='$*'
}
get_sanitized(){
    eval echo \$$(sanitize_var_name "$1")
}

sanitize_var_name "yyuj@_cflags"

set_sanitized "yyuj@_cflags" I/mingw32/SDL2 -Dmain=SDL_main

echo $yyuj__cflags

get_sanitized "yyuj@_cflags"
```

运行结果如下：

![configure-function2-1-1](configure-function2\configure-function2-1-1.png)

可以看到， `@` 这种特殊字符被替换成 `_`，这条 sed 的正则 `'/[^A-Za-z0-9_]/_/g` 就是把不是字母跟数字的字符全部替换成 `_` 。



------

**2，pushvar，popvar**，这两个函数也是一组的，但是没有地方使用到这两个函数，应该是遗弃的函数忘记删除了。

------

**3，request** ，这个函数是把后缀为 `_requested` 的变量设置为 yes，批量设置的，示例代码如下：

```
#!/bin/sh
EXTERNAL_AUTODETECT_LIBRARY_LIST="
    alsa
    lzma
    mediafoundation
    schannel
    sdl2
    securetransport
    sndio
    xlib
    zlib
"

request(){
    for var in $*; do
        eval ${var}_requested=yes
        eval $var=
    done
}

for lib in $EXTERNAL_AUTODETECT_LIBRARY_LIST; do
    request $lib
done

echo "zlib_requested="$zlib_requested
echo "xlib_requested="$zlib_requested
```

运行结果如下：

![configure-function2-1-2](configure-function2\configure-function2-1-2.png)

------

**4，warn_if_gets_disabled**，一个普通的函数，设置一个变量 WARN_IF_GETS_DISABLED_LIST 。

------

**5，disable_with_reason**，把变量设置成 no，变量通常是一个库的名字，如果这个库是 requested （必须的），直接 die 退出。

```
disable_with_reason(){
    disable $1
    eval "${1}_disable_reason=\"$2\""
    if requested $1; then
        die "ERROR: $1 requested, but $2"
    fi
}
```

------

**6，enable_sanitized，disable_sanitized**，这两个函数 里面调了 enable 函数，所以是类似的，只是会对变量名做转义例如 @ 换成 _ 。

------

**7，do_enable_deep，enable_deep，enable_deep_weak**，这 3个函数是一组的，主要作用是 递归调用，不断设置下级的 属性成 yes，所以示例代码比较多，示例代码如下：

```
#!/bin/sh
set_all(){
    value=$1
    shift
    for var in $*; do
        eval $var=$value
    done
}

enable(){
    set_all yes $*
}
set_weak(){
    value=$1
    shift
    for var; do
        eval : \${$var:=$value}
    done
}
enabled(){
    test "${1#!}" = "$1" && op="=" || op="!="
    eval test "x\$${1#!}" $op "xyes"
}

disabled(){
    test "${1#!}" = "$1" && op="=" || op="!="
    eval test "x\$${1#!}" $op "xno"
}

enable_weak(){
    set_weak yes $*
}

do_enable_deep(){
    for var; do
        enabled $var && continue
        set -- $var
        eval enable_deep \$${var}_select
        var=$1
        eval enable_deep_weak \$${var}_suggest
    done
}

enable_deep(){
    do_enable_deep $*
    enable $*
}

enable_deep_weak(){
    for var; do
        disabled $var && continue
        set -- $var
        do_enable_deep $var
        var=$1
        enable_weak $var
    done
}

ffmpeg_select="aformat_filter anull_filter atrim_filter format_filter
               hflip_filter null_filter
               transpose_filter trim_filter vflip_filter"
ffmpeg_suggest="ole32 psapi shell32"

aformat_filter_select="test33 test44 test55"


enable_deep_weak $ffmpeg_select $ffmpeg_suggest

echo "aformat_filter="$aformat_filter
echo "test33="$test33
echo "test44="$test44

```

运行结果如下：

![configure-function2-1-3](configure-function2\configure-function2-1-3.png)

上面的 3 个函数，只有 `enable_deep_weak` 函数被外部使用，所以我只演示了 `enable_deep_weak` 的时候，可以看到 ，我没有传递 `aformat_filter_select` 变量给 `enable_deep_weak` 函数，但是 test_33 依然被设置成 yes 了，这是因为  `ffmpeg_select` 变量里面有个 `aformat_filter` 字符串，循环里面会自动拼接上 `_select` 后缀来取到 变量 `aformat_filter_select` 来遍历。

shell 这门语言实际上不适合做太复杂的循环逻辑。因为没法断点调试。

------

**8，requested，enabled，disabled，enabled_all，disabled_all，enabled_any，disabled_any**，这 7 个函数是一组的。

configure 脚本里面有 `enable` 跟 `enabled` 函数，`request` 跟 `requested`，d 后缀 代表这是一个判断函数，没有 d 后缀代表这是一个 赋值函数。

all 代表全部都是 enable 才返回 1，any 代表只要有一个 enable 就返回 1。

------

**9，set_default** ，这是一个设置默认值函数，如果变量没设置，就设置成默认值，默认值是一个 `_default` 后缀的东西。

------

**10，is_in**，判断一个字符串是不是在一个数组里面，示例代码如下：

```
#!/bin/sh
is_in(){
    value=$1
    shift
    for var in $*; do
        [ $var = $value ] && return 0
    done
    return 1
}

ARCH_LIST="
    arm
    x86
    x86_32
    x86_64
"
arch="x86"
is_in $arch $ARCH_LIST || echo "unknown architecture $arch"
arch="x87"
is_in $arch $ARCH_LIST || echo "unknown architecture $arch"
```

运行结果如下：

![configure-function2-1-4](configure-function2\configure-function2-1-4.png)

可以看到，x87 不在数组里面就会提示错误。

------

**11，check_deps，**这是一个检测库依赖 的函数，里面会递归调 自己 （check_deps） ，也会调 `enable_deep_weak` ，所以有数千次循环。

`check_deps` 这个函数比较复杂，他的内部逻辑如下，我会用  abcd 来标示步骤：

------

**a，**通过一个 xxx_checking 变量来避免死循环，如下：

提示：刚开始的 `x\` 里面的 x 就是一个字母而已， `\` 是转义后面的 `$` 的。

![configure-function2-1-5](configure-function2\configure-function2-1-5.png)

------

**b，**然后他有 7 种依赖选项，cfg 代表一个变量，如下：

![configure-function2-1-6](configure-function2\configure-function2-1-6.png)

我用一些实际的例子讲一下 上面这 7 种依赖是什么意思？首先大家可以在 configure 里面看到下面这样的代码：

```
ffmpeg_deps="avcodec avfilter avformat"
ffmpeg_select="aformat_filter anull_filter atrim_filter format_filter
               hflip_filter null_filter
               transpose_filter trim_filter vflip_filter"
ffmpeg_suggest="ole32 psapi shell32"
```

从上面代码可以看出，ffmpeg 这个命令行程序他依赖 3 个基本库 `avcodec` ，`avfilter`，`avformat`，如果这 3 个库检测不通过， ffmpeg 程序就不会编译出来。

然后 shell 脚本是怎么检测 `avcodec` 库的呢？

是直接通过 enabled 函数判断，实际上就是判断 `$avcodec` 变量是不是 yes。因此 只有 `$avcodec`，`$avfilter`，`$avformat` 这 3 个变量都是 yes，才会把 `$ffmpeg` 变量设置成 `yes`。

------

因此  `check_deps` 函数的作用就比较明显了，就是检测 程序 或者 库的依赖，根据 依赖来设置成 yes。如果某个依赖项是 no，就会报错退出。

但是 `check_deps` 里面有 7 种依赖，我们上面只讲了 deps 后缀的依赖。7 种依赖如下，xxx代表一个变量

- cfg_deps，必须的依赖，依赖项必须全部是 yes，如果有一个是 no， 直接调 `disable_with_reason` 提示错误，然后退出。
- cfg_deps_any，只要其中一个依赖项 是 yes 即可。
- cfg_conflict，跟某些项 冲突，里面所有的项（item） 必须是 no，要不会提示错误退出。
- cfg_select，这个变量应该是可以让用户自定义的，但是里面的用法优势 `disabled_any $dep_sel` 只要有一个 是 no 就会报错，暂时感觉跟  xxx_deps 是一样的。不过应该是某些情况 xxx_select 会根据用户自定义的参数进行变换，也就是根据  configure的参数 变换。

- cfg_suggest，这是一个建议选项，例如 `ffmpeg_suggest="ole32 psapi shell32"`，一旦 `$ffmpeg` 设置为 `yes`，`$ole32` 也会尽量设置成 `yes`，因为用的是 weak 设置，如果之前 `$ole32` 是 no， 也不会报错，因为不会做依赖检测，只是建议使用。
- cfg_if，如果 `$cfg_if` 里面的变量全都是 yes，就会尽量设置 `$cfg` 变量为 yes。
- cfg_if_any，如果 `$cfg_if_any` 里面的变量有一个是 yes，就会尽量设置 `$cfg` 变量为 yes。

因此 check_deps 就是通过依赖配置来设置一些变量 成 yes，如果不能依赖的项是 no，这又是必须的 依赖，就会报错。

------

 `check_deps` 函数里面有以下一句注释，代表大部分的 cfg 都是没有依赖的，但是循环了很多次，需要优化这部分的代码。

> most of the time here $cfg has no deps - avoid costly no-op work

------

`check_deps` 函数最后的一句代码也比较关键，如下：

```
enabled $dep && eval append ${cfg}_extralibs ${dep}_extralibs
```

因为 cfg 依赖 dep，所以这句代码 把 dep 依赖的外部库，全部加进去 ${cfg}_extralibs。

静态库的依赖应该也是这句代码搞的，例如 ffmpeg  依赖 libavcodec.a，而 libavcodec.a 依赖 mfplat.lib mfuuid.lib ole32.lib 等等的静态库。

------

至此，`check_deps` 函数分析完毕。



