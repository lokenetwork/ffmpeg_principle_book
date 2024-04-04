# configure函数分析-C章—FFFmpeg configure源码分析

configure 里面定义了 几十个 shell 函数，本文就用示例代码来介绍这些函数的具体用法。

------

**1，print_config**，这个函数的作用是把 shell 的那些 yes 跟 no 变量转存到各种文件里面。示例代码如下：

```
#!/bin/sh

#创建 test.h 文件演示。
touch test.h
touch test.mak

map(){
    m=$1
    shift
    for v; do eval $m; done
}

print_config(){
    pfx=$1
    files=$2
    shift 2
    map 'eval echo "$v \${$v:-no}"' "$@" |
    awk "BEGIN { split(\"$files\", files) }
        {
            c = \"$pfx\" toupper(\$1);
            v = \$2;
            sub(/yes/, 1, v);
            sub(/no/,  0, v);
            for (f in files) {
                file = files[f];
                if (file ~ /\\.h\$/) {
                    printf(\"#define %s %d\\n\", c, v) >>file;
                } else if (file ~ /\\.asm\$/) {
                    printf(\"%%define %s %d\\n\", c, v) >>file;
                } else if (file ~ /\\.mak\$/) {
                    n = -v ? \"\" : \"!\";
                    printf(\"%s%s=yes\\n\", n, c) >>file;
                } else if (file ~ /\\.texi\$/) {
                    pre = -v ? \"\" : \"@c \";
                    yesno = \$2;
                    c2 = tolower(c);
                    gsub(/_/, \"-\", c2);
                    printf(\"%s@set %s %s\\n\", pre, c2, yesno) >>file;
                }
            }
        }"
}

config_files="test.h test.mak"

ARCH_LIST="
    aarch64
    alpha
    arm
    x86
    x86_32
    x86_64
"
print_config ARCH_  "$config_files" $ARCH_LIST
```

运行 之后查看 test.h 跟  test.mak 文件，内容如下：

![configure-function3-1-1](configure-function3\configure-function3-1-1.png)

可以看到  print_config 实际上就是把 一些变量配置 存到 .h  跟 .mak 文件，yes 会转成 1，no 会转成 0。他里面用了 split 函数来分割出来文件名循环处理。



------

**2，print_enabled**，打印出值是 yes 的变量，变量通常是一个库名，示例代码如下：

```
#!/bin/sh
enabled(){
    test "${1#!}" = "$1" && op="=" || op="!="
    eval test "x\$${1#!}" $op "xyes"
}

openal="yes"
opengl="yes"

EXTERNAL_LIBRARY_LIST="
    libxml2
    libzimg
    libzmq
    libzvbi
    openal
    opengl
"

print_enabled(){
    suf=$1
    shift
    for v; do
        enabled $v && printf "%s\n" ${v%$suf}
    done
}

echo "External libraries:"
print_enabled '' $EXTERNAL_LIBRARY_LIST
```

运行结果如下：

![configure-function3-1-2](configure-function3\configure-function3-1-2.png)

------

**3，append ，prepend**，这两个函数是把往变量添加数据，一个加在后面，一个加在前面，用空格隔开加进来的各个值

------

**4，reverse**，反转一个数组，示例如下：

```
#!/bin/sh
reverse () {
    eval '
        reverse_out=
        for v in $'$1'; do
            reverse_out="$v $reverse_out"
        done
        '$1'=$reverse_out
    '
}

EXTERNAL_LIBRARY_LIST="
    libxml2
    libzimg
    libzmq
    libzvbi
    openal
    opengl
"
echo $EXTERNAL_LIBRARY_LIST
reverse EXTERNAL_LIBRARY_LIST
echo $EXTERNAL_LIBRARY_LIST
```

运行结果如下：

![configure-function3-1-3](configure-function3\configure-function3-1-3.png)

------

**5，unique**，去除数组里面的重复项，保留最后一个。示例代码如下：

```
#!/bin/sh
reverse () {
    eval '
        reverse_out=
        for v in $'$1'; do
            reverse_out="$v $reverse_out"
        done
        '$1'=$reverse_out
    '
}

unique(){
    unique_out=
    eval unique_in=\$$1
    reverse unique_in
    for v in $unique_in; do
        # " $unique_out" +space such that every item is surrounded with spaces
        case " $unique_out" in *" $v "*) continue; esac  # already in list
        unique_out="$unique_out$v "
    done
    reverse unique_out
    eval $1=\$unique_out
}

EXTERNAL_LIBRARY_LIST="
    libzmq
    libzvbi
    openal
    opengl
    libzvbi
"

echo $EXTERNAL_LIBRARY_LIST
unique EXTERNAL_LIBRARY_LIST
echo $EXTERNAL_LIBRARY_LIST
```

运行结果如下：

![configure-function3-1-4](configure-function3\configure-function3-1-4.png)

------

**6，add_cppflags**，往变量 `$CPPFLAGS` 里面加数据

------

**7，add_cflags，add_cflags_headers，add_cxxflags，add_objcflags，add_host_ldflags**，这些函数都是往一个变量 append 数据，但是他们使用了 xxx_cflags_filter 来转换参数，我举其中一个例子来讲解这个 转换参数的逻辑，示例代码如下：

```
#!/bin/sh
append(){
    var=$1
    shift
    eval "$var=\"\$$var $*\""
}
msvc_flags(){
    for flag; do
        case $flag in
            -Wall)                echo -W3 -wd4018 -wd4146 -wd4244 -wd4305     \
                                       -wd4554 ;;
            -Wextra)              echo -W4 -wd4244 -wd4127 -wd4018 -wd4389     \
                                       -wd4146 -wd4057 -wd4204 -wd4706 -wd4305 \
                                       -wd4152 -wd4324 -we4013 -wd4100 -wd4214 \
                                       -wd4307 \
                                       -wd4273 -wd4554 -wd4701 -wd4703 ;;
        esac
    done
}
cflags_filter="msvc_flags"
CFLAGS="666 "
add_cflags(){
    append CFLAGS $($cflags_filter "$@")
}
add_cflags -Wall
echo "CFLAGS="$CFLAGS
```

运行结果如下：

![configure-function3-1-5](configure-function3\configure-function3-1-5.png)

因此 `xxx_cflags_filter` 这种函数，就是把一些选项做转换，因为 FFmpeg 是支持多种编译器链接器的，每个编译器选项不一样，所以需要转换一下。

例如 把 `-lz` 转成 `zlib.lib`，把 `-lx264` 转成 `libx264.lib`

------

**8，add_compat** ，往 compat_objs 变量添加数据，同时可以定义一些宏，compat_objs 变量通常是一些运行时库，每个系统的一些运行时库不太一样。示例代码如下：

```
#!/bin/sh
append(){
    var=$1
    shift
    eval "$var=\"\$$var $*\""
}
add_cppflags(){
    append c "$@"
}
map(){
    m=$1
    shift
    for v; do eval $m; done
}
append(){
    var=$1
    shift
    eval "$var=\"\$$var $*\""
}

add_compat(){
    append compat_objs $1
    shift
    map 'add_cppflags -D$v' "$@"
}

echo "compat_objs="$compat_objs
echo "c="$c

add_compat msvcrt/snprintf.o
add_compat strtod.o strtod=avpriv_strtod

echo "compat_objs="$compat_objs
echo "c="$c
```

运行结果如下：

![configure-function3-1-6](configure-function3\configure-function3-1-6.png)

------

**9，test_cmd**，执行一个命令，并把输出结果保存进去 logfile ，然后返回值会被用了判断。

------

**10，test_stat**，保存文件信息进去 logfile 

