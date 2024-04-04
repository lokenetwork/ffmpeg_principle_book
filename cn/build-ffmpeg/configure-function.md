# configure函数分析-A章—FFFmpeg configure源码分析

configure 里面定义了 几十个 shell 函数，本文就用示例代码来介绍这些函数的具体用法。有些函数只给了函数名，没贴代码是为了节省篇幅。

------

**1，show_help** ，这是一个输出帮助信息的函数。他没有简单的 echo 字符串来输出字符串，而是使用了 cat 命令。

这样做有什么好处呢？因为 cat 可以预定义格式，就想 html 的 `<pre>` 便签一样。

cat 命令的通常用法是查看一个文件的内容，如下：

```
cat RELEASE_NOTES
```

![configure-function-1-1](configure-function\configure-function-1-1.png)

但是也可以直接 用 `<<` 符号输入预定义的格式内容给 cat 来输出，如下：

```
cat <<EOF
Usage: configure [options]
Options: [defaults in brackets after descriptions]

Help options:
  --help                   print this message
  --quiet    
NOTE: Object files are built at the place where configure is launched.
EOF
```

![configure-function-1-2](configure-function\configure-function-1-2.png)

这样输出的信息就是有格式的，非常方便。show_help 函数就是用的这种方法。



------

**2，log**，输出信息到日志文件。在 configure 的定义如下，非常简单，不做讲解：

```
log(){
    echo "$@" >> $logfile
}
```

------

**3，log_file**，这是 一个把代码文件的内容 记录进去日志文件的函数。

我们可以建一个 test-configure 的脚本，赋予执行的权限，测试一下 log_file 这个函数，代码如下：

```
#!/bin/sh
logfile=mylog.txt
log(){
    echo "$@" >> $logfile
}

log_file(){
    log BEGIN "$1"
    log_file_i=1
    while IFS= read -r log_file_line; do
        printf '%5d\t%s\n' "$log_file_i" "$log_file_line"
        log_file_i=$(($log_file_i+1))
    done < "$1" >> "$logfile"
    log END "$1"
}
log_file test.c
```

在相同的目录创建一个 test.c 代码文件，内容如下：

```
#include <SDL_events.h>
#include <stdint.h>
long check_SDL_PollEvent(void) { return (long) SDL_PollEvent; }
int main(void) { 
	int ret = 0;
	ret |= ((intptr_t)check_SDL_PollEvent) & 0xFFFF;
	return ret; 
}
```

然后执行以下 `test-configure`  命令测试一下 log_file 函数。可以看到，test.c 里面的代码全都记录进入 mylog.txt 文件了，如下：

![configure-function-1-3](configure-function\configure-function-1-3.png)

FFmpeg 为什么要创建一个 log_file 函数来记录代码，这是因为 configure 脚本检测环境使用的方式就是实际地编译一遍，这是最靠谱的方法，上面 test.c 的代码是直接把 sdl 的头文件编译进来，然后调一个 sdl 函数，编译通过那就是没有问题。

而 FFmpeg 有很多的库需要检测，他们是共用一个 test.c ，例如 sdl 的 test.c 编译过之后，test.c 内容就会被 nvcc 硬件编解码的代码覆盖。所以需要把每个阶段的 test.c 记录进去日志，方便用户排查编译错误的问题。

------

**4，warn**，警告函数，die 退出函数，cat 输出一堆信息之后 调 exit 1 退出。

------

**5，第 5 部分是字符串处理函数。**

- toupper，把字符串转大小。
- tolower，把字符串转成小写。
- c_escape，转义函数，好像是把 `"` 或者 `\` 字符替换成 `\0`。我没找到具体的字符串来演示这个函数，感觉是一些特别特殊的场景才用得上这个 c_escape
- sh_quote，暂时没看懂这个函数的用法，后面补充。
- cleanws，不知道干嘛

上面这些函数经常用到 `$@` 跟 `$*` ，这两个变量的用法请看 《[Shell特殊变量](http://c.biancheng.net/view/806.html)》

上面这些函数 在 configure 里面的用法如下：

```
#!/bin/sh
c_escape(){
    echo "$*" | sed 's/["\\]/\\\0/g'
}
sh_quote(){
    v=$(echo "$1" | sed "s/'/'\\\\''/g")
    test "x$v" = "x${v#*[!A-Za-z0-9_/.+-]}" || v="'$v'"
    echo "$v"
}

for v in "$@"; do
    r=${v#*=}
    echo $r
    l=${v%"$r"}
    echo $l
    r=$(sh_quote "$r")
    FFMPEG_CONFIGURATION="${FFMPEG_CONFIGURATION# } ${l}${r}"
done

echo $FFMPEG_CONFIGURATION
```

运行结果如下：

```
./test-configure  --prefix=/home/ubuntu/ffmpeg/build64/ffmepg-4.4-ubuntu      --enable-gpl
```

![configure-function-1-4](configure-function\configure-function-1-4.png)

上面这些 字符串处理函数里面有很多正则，我也没细看，反正就是处理一些特殊字符，转义，然后输出正规的字符串，上图中，我敲多了很多空格 在 --enable-gpl 前面，经过处理之后变成一个空格了。

------

**6，filter，filter_out**，这两个函数其实是用来分离 编译器跟链接器的选项的，我写一个例子你就明白了，如下：

```
#!/bin/sh
filter(){
    pat=$1
    shift
    for v; do
        eval "case '$v' in $pat) printf '%s ' '$v' ;; esac"
    done
}

filter_out(){
    pat=$1
    shift
    for v; do
        eval "case '$v' in $pat) ;; *) printf '%s ' '$v' ;; esac"
    done
}

test_ld(){
    shift 1
    flags=$(filter_out '-l*|*.so' $@)
    echo "flags is "$flags
    libs=$(filter '-l*|*.so' $@)
    echo "libs is "$libs
}
test_ld cc -lm sdl2.so -I/usr/local/include/nv_sdk_10.1
```

上面的代码有两个重点。

- shift，代表移动 `$` 变量。就是 $2 会变成 $1 ，$3 变成 $2，以此类推。
- `for v; do`，这种 for 循环写法，没有指定从那个数组遍历，默认就是遍历 $1 ~ $末尾变量。

运行结果如下：

![configure-function-1-5](configure-function\configure-function-1-5.png)

从上图可以看出， `-I` 选项是 编译器的选项，指定哪个目录找头文件。而 `-l`  跟 so 这些 是链接器才会用得到。

------

**7，map** ，批量赋值函数。

```
#!/bin/sh
ARCH_EXT_LIST_ARM="
    armv5te
    armv6
    armv6t2
    armv8
    neon
    vfp
    vfpv3
    setend
"

map(){
    m=$1
    shift
    for v; do eval $m; done
}

map 'eval ${v}_inline_deps=inline_asm' $ARCH_EXT_LIST_ARM

echo $armv5te_inline_deps
echo $armv6_inline_deps
```

运行结果如下：

![configure-function-1-6](configure-function\configure-function-1-6.png)

所以 map 函数实际上是一个批量赋值函数，把数组里面各个元素替换给 `$v` ，然后进行赋值。 configure 有好几个地方使用了 map 函数。

------

**8，add_suffix**，批量添加后缀函数。

```
#!/bin/sh
ARCH_EXT_LIST_ARM="
    armv5te
    armv6
    armv6t2
    armv8
    neon
    vfp
    vfpv3
    setend
"

add_suffix(){
    suffix=$1
    shift
    for v; do echo ${v}${suffix}; done
}

add_suffix _external $ARCH_EXT_LIST_ARM
```

运行结果如下：

![configure-function-1-7](configure-function\configure-function-1-7.png)

------

**9，remove_suffix**，批量删除后缀函数，用的是 ` ${v%$suffix}` 这种字符串替换法，跟 `add_suffix` 函数类似，就不演示了。代码如下：

```
remove_suffix(){
    suffix=$1
    shift
    for v; do echo ${v%$suffix}; done
}
```

------

**10，set_all ，enable ，disable**，这 3个函数是一起使用的，可以批量设置一些变量的值为 yes 或者 no。使用示例如下：

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
disable(){
    set_all no $*
}

enable loongson2 loongson3

echo "loongson2="$loongson2
echo "loongson3="$loongson3
```

运行结果如下：

![configure-function-1-8](configure-function\configure-function-1-8.png)

------

**11，set_weak** ，set_weak 跟 set_all 其实是一组兄弟函数，只是 `set_weak` 是弱赋值，定义如下：

```
set_weak(){
    value=$1
    shift
    for var; do
        eval : \${$var:=$value}
    done
}
```

之前已经讲过 `:=` 这种赋值法，就是这个变量如果已经定义了，就不赋值。

