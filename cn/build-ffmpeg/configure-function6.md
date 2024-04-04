# configure函数分析-E章—FFFmpeg configure源码分析

从这章开始，是一些扩展函数的讲解。

------

**1，find_things_extern** ，查找 C 代码里面的 extern 后面的字符串。示例代码如下：

```
#!/bin/sh
source_path="/home/ubuntu/Documents/ffmpeg-4.4.1"
find_things_extern(){
    thing=$1
    pattern=$2
    file=$source_path/$3
    out=${4:-$thing}
    echo "out="$out
    sed -n "s/^[^#]*extern.*$pattern *ff_\([^ ]*\)_$thing;/\1_$out/p" "$file"
}

find_things_extern bsf AVBitStreamFilter libavcodec/bitstream_filters.c
```

运行结果如下：

![configure-function6-1-2](configure-function6\configure-function6-1-2.png)

![configure-function6-1-1](configure-function6\configure-function6-1-1.png)

从上图可以看出，就是用 sed 命令在 bitstream_filters.c 文件里面查找符合正则的字符串。

FFmpeg 这么搞的好处是，只需要写一个地方的代码，其他地方要用，就 用 sed 提取字符串。

------

**2，find_filters_extern**，也是提取 C 代码里面的字符串，示例代码如下：

```
#!/bin/sh
source_path="/home/ubuntu/Documents/mix"
find_filters_extern(){
    file=$source_path/$1
    sed -n 's/^extern AVFilter ff_[avfsinkrc]\{2,5\}_\([[:alnum:]_]\{1,\}\);/\1_filter/p' $file
}

find_filters_extern libavfilter/allfilters.c
```

运行结果如下：

![configure-function6-1-3](configure-function6\configure-function6-1-3.png)

![configure-function6-1-4](configure-function6\configure-function6-1-4.png)

------

**3，print_in_columns**，这是一个输出信息函数，示例代码如下：

```
#!/bin/sh
LIBRARY_LIST="
    avdevice
    avfilter
    swscale
    postproc
    avformat
    avcodec
    swresample
    avresample
    avutil
"
avdevice="yes"
avfilter="yes"
swscale="yes"
avformat="yes"

enabled(){
    test "${1#!}" = "$1" && op="=" || op="!="
    eval test "x\$${1#!}" $op "xyes"
}

print_enabled(){
    suf=$1
    shift
    for v; do
        enabled $v && printf "%s\n" ${v%$suf}
    done
}

print_in_columns() {
    tr ' ' '\n' | sort | tr '\r\n' '  ' | awk -v col_width=24 -v width="$ncols" '
    {
        num_cols = width > col_width ? int(width / col_width) : 1;
        num_rows = int((NF + num_cols-1) / num_cols);
        y = x = 1;
        for (y = 1; y <= num_rows; y++) {
            i = y;
            for (x = 1; x <= num_cols; x++) {
                if (i <= NF) {
                  line = sprintf("%s%-" col_width "s", line, $i);
                }
                i = i + num_rows;
            }
            print line; line = "";
        }
    }' | sed 's/ *$//'
}

echo "Libraries:"
print_enabled '' $LIBRARY_LIST | print_in_columns
echo
```

运行结果如下：

![configure-function6-1-5](configure-function6\configure-function6-1-5.png)

`configure` 往控制台输出信息就是用的 `print_in_columns` 函数

------

**4，show_list**，也是一个输入信息到控制台的函数，不用特别在意。

------

**5，rand_list，do_random**，这两个函数好像是 FFmpeg 开发者为了测试一些依赖有没问题，可以随机启用一些库，来测试 configure的代码有无问题。所以这两个应该是测试函数

------

**6，tmpfile**，创建临时文件函数。

------

**7，probe_cc**，探测编译器函数，非常重要。这个函数里面主要设置两个变量， `${pfx}_type` 跟  `${pfx}_ident` 如下：

![configure-function6-1-6](configure-function6\configure-function6-1-6.png)

------

**8，set_ccvars**，设置编译器的一些变量，里面的 `_DEPCMD` 我也不太清楚，埋个坑，后面补充

------

**9，probe_libc**，探测 C 语言的运行时。如下：

![configure-function6-1-7](configure-function6\configure-function6-1-7.png)

主要设置一些 编译器选项，还有确定 C 语言的运行时类型 `libc_type`
