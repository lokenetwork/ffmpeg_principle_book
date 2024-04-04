# configure函数分析-C章—FFFmpeg configure源码分析

configure 里面定义了 几十个 shell 函数，本文就用示例代码来介绍这些函数的具体用法。有些函数只给了函数名，没贴代码是为了节省篇幅。

------

**1，cc_e，cc_o，as_o，x86asm_o，ld_o，hostcc_e，nvcc_o** ，这些函数的功能都是类似的，就是往一个参数前面加选项，示例代码如下：

```
#!/bin/sh
CC_E='-E -o $@'
CC_O='-o $@'
cc_e(){
    eval printf '%s\\n' $CC_E
}
cc_o(){
    eval printf '%s\\n' $CC_O
}

cc_e test.o
cc_o test.o
```

运行结果如下：

![configure-function4-1-1](configure-function4\configure-function4-1-1.png)

可以看到，前面加了 `-E` 跟 `-o` 这些其实都是编译器的选项。



------

**2，test_cc**，这个函数是重点，这个函数就是尝试编译一个文件。编译通过代表环境没问题。

`test_cc` 函数里面 的 `cat > $TMPC` 比较难懂， cat 这里没有输入文件，那什么是输入呢？

通常会用 `<<` 符号输入内容给 test_cc 命令，然后 cat 就有输入了。示例代码如下：

```
#!/bin/sh
logfile="mylog.txt"
TMPC="test.c"
CC_O='-o $@'

set_all(){
    value=$1
    shift
    for var in $*; do
        eval $var=$value
    done
}

disable(){
    set_all no $*
}

test_cmd(){
    log "$@"
    "$@" >> $logfile 2>&1
}

cc_o(){
    eval printf '%s\\n' $CC_O
}


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


test_cc(){
    log test_cc "$@"
    cat > $TMPC
    log_file $TMPC
    test_cmd $cc $CPPFLAGS $CFLAGS "$@" $CC_C $(cc_o $TMPO) $TMPC
}

test_cc <<EOF || disable nvenc
#include <ffnvcodec/nvEncodeAPI.h>
NV_ENCODE_API_FUNCTION_LIST flist;
void f(void) { struct { const GUID guid; } s[] = { { NV_ENC_PRESET_HQ_GUID } }; }
int main(void) { return 0; }
EOF

echo "nvenc="$nvenc;
```

运行结果如下：

![configure-function4-1-2](configure-function4\configure-function4-1-2.png)

上面的代码 test.c 编译报错，就会把 nvenc 设置 no。

这其实就是 FFmpeg 检测外部库有没问题的方法，就是实实在在的编译一次，把一些头文件，函数调用一次。

------

**3，test_cxx**，这个函数跟 test_cc 函数是几乎一样的，只是多了一个 `$CXXFLAGS` 选项，如下：

![configure-function4-1-3](configure-function4\configure-function4-1-3.png)

为什么有 test_cxx ，FFmpeg 自己的代码都是 C 代码，这应该是有可能 外部的库有些是 C++ 写的。

------

**4，test_objcc**，这个函数是编译 .m 后缀的文件的，应该是 苹果系统编译会用到，我暂时不太熟悉，后面再补充。

------

**5，test_nvcc**，编译 .cu 后缀的文件，cuda 测试。

------

**6，check_nvcc**，检测 nvcc 环境，里面调了 test_nvcc 。

------

**7，test_cpp**，也是测试一下能否编译通过一个 cpp 文件。

------

**8，test_as，test_x86asm**，测试一下能否编译通过一个汇编文件。FFmpeg 里面有很多地方用了汇编优化。

------

9，check_cmd ，这是一个没用的函数，没地方使用。

------

**10，check_as**，这个函数比较重要，但是又比较复杂，不太好演示，依赖的地方有点多，我直接贴代码讲解，如下：

![configure-function4-1-4](configure-function4\configure-function4-1-4.png)

整个逻辑是这样的， `.fpu neon` 就是汇编代码。尝试用 `test_as` 编译这段汇编代码，如果编译通过就调 `enable` 来设置 `$as_fpu_directive` 变量成 yes。

------

**11，check_inline_asm**，这个函数是尝试编译一段嵌入到 C 程序的汇编，如果编译通过，就把 `$1` 变量设置为 yes。

------

**12，check_inline_asm_flags**，这个也是尝试编译一段嵌入到 C 程序的汇编，但是编译通过之后，还会加一些 flags，如下：

![configure-function4-1-5](configure-function4\configure-function4-1-5.png)

------

**13，check_insn，**同时测试 嵌入C的汇编 跟 纯汇编，里面调了 `check_inline_asm` 跟 `check_as`

------

到这里，需要提一个重点，configure 里面有很多很多的函数，但是他们的目的，其实都是测试某些东西，然后测试通过后，设置某些变量成 yes。

------

**14，test_ld**，尝试链接生成一个可执行文件，如果生成成功，这个命令返回 0 ，生成失败 返回 1，调用者通常会根据返回值把 某些变量设置成 yes。用法如下：

![configure-function4-1-6](configure-function4\configure-function4-1-6.png)

------

**15，check_ld**，里面调了 test_ld，根据 test_ld 返回值设置某些变量成 yes。

------

**16，print_include**，针对不同写法的头文件做下处理

```
#!/bin/sh
print_include(){
    hdr=$1
    test "${hdr%.h}" = "${hdr}" &&
        echo "#include $hdr"    ||
        echo "#include <$hdr>"
}
print_include test leo.h
print_include leo.h
```

运行结果如下：

![configure-function4-1-7](configure-function4\configure-function4-1-7.png)

------

**17，test_code**，里面可能会调 `test_cc`，我为什么说可能，因为他调的是 `test_$1` ，`$1` 是由外部传进来的，可能是 cc 也可能是 其他的 。如果 `$1` 是 cc 那就相当于 对 `test_cc` 做了一些简单的封装。

------

**18，check_cppflags，check_cflags，check_cxxflags，check_ldflags**，这些函数都是类似的，都是检测编译器选项有无问题，没问题就 加进去 CPPFLAGS 变量 或者其他的变量。

------

**19，check_stripflags** ，这个函数有点特殊，是检测 编译器选项 的那个 strip 选项，删除调试信息。

------

**20，check_headers**，这个函数比较重要，实际上即使检测一下一些头文件是否存在，因为只有存在才会编译通过，通过就会设置为 yes。请看下图

![configure-function4-1-8](configure-function4\configure-function4-1-8.png)

如果编译通过 **$sys_resource_h** 变量就会设置为 yes， 因为他用了 sanitized 转义变量名。

------

**21，check_header_objcc** ，也是一个用于苹果系统的函数，暂时不讲解。

------

22，check_apple_framework，也是一个用于苹果系统的函数，暂时不讲解。

