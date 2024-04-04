# configure语法摘要—FFFmpeg configure源码分析

我一直没想好怎么来写 configure 分析这一章节的内容。因为 FFmpeg 的 configure 脚本有 8千行代码，而且这些是 shell 代码，不太方便断点调试，非常复杂。

我意识到 如果 逐行 分析 这 8千行代码，不利于读者阅读。逐行分析 跟 直接给一个 带注释的 configure 脚本是一样的。

所以 configure 这一章，会拆成几个部分来讲解。因为基本上任何代码，都有模块的概念的，一个一个模块分析 是比较好的方式。

------

Shell 的基本知识推荐一本书[《Linux命令行与shell脚本编程大全》](https://union-click.jd.com/jdc?e=618%7Cpc%7C&p=JF8BANIJK1olXDYCV19dCUgQBF9MRANLAjZbERscSkAJHTdNTwcKBlMdBgABFksUAm8JGFwSQl9HCANtUz13dhMIelx1XkZaCAEtSypWGWhVTVcZbQcyV19dCE4fBm8OGGslXQEyAjBdCUoWAm4NH1wSbQcyVFlZDUMSA2cKHlkXWzYFVFdtUx55Bm9dGQsRWAVXUF0OXXsnM2w4HFscSQBwFQxJDjknM284GGtXM1VVAw5VXB9AVjwLT14QXAUKAQ5dDE4VVz8NH1oTWAQLZFxcCU8eM18)

------

由于 configure 是 shell 脚本，而 shell 的语法有很多种，假设 shell 的语法有 1 万种，但是 configure 里面只用了 200 种语法。

所以本文先介绍 configure 里面用到的 一些 shell 的基本语法，这些语法学了立马就能用得上的。

**可以把本文作为一个 shell 手册查询，不需要从头开始阅读本文，当遇到 configure 的语法没看懂时可以回过头看本文。**

------

1，export ，导出（设置）一个环境变量，在 configure 里面的用法如下：

```
LC_ALL=C
export LC_ALL
```

大家可以在 Linux 终端 按顺序 执行一下 下面的命令。

```
echo $LC_ALL
export LC_ALL=C
echo $LC_ALL
```

![configure-syntax-1-1](configure-syntax\configure-syntax-1-1.png)

从上图可以看出来，刚开始 echo 变量 LC_ALL 的时候是没有内容的，export 之后就有内容了。这个 export 实际上就是设置一个变量的值，但是只在当前终端会话有效，你关闭这个终端再打开这个 LC_ALL 变量就会重新变成没有内容的情况。

至于这个 LC_ALL 变量是干什么的，推荐阅读 这篇文章[《shell脚本中 LC_ALL=C 的含义》](https://blog.csdn.net/happyhorizion/article/details/80529301)。

------

在 shell 变量里面还有一种也比较常用的设置变量的方式，就是两条命令连一起，这样 设置的变量只对后面的命令生成，对终端会话不生效。代码如下：

TODO：这里找一下 configure 的应用，应该有。



------

2，unset ，这个是删除一个变量，例如 `unset foo`

------

3，`${foo%%bar}`  ，这是一种删除变量最后字符串的用法，示例代码如下：

```
foo=lsbar
echo ${foo%%bar}
```

![configure-syntax-1-2](configure-syntax\configure-syntax-1-2.png)

如上，%% 会把 最后的 bar 字符删除。

所以我为什么要讲一些语法，是因为 shell 这些语法真的有点奇葩，: %% 这种符号语法 用搜索引擎真的很难搜到说明用法，去翻 shell 手册，又要翻很久。

------

4，test ，这个是测试命令，可以判断各种条件跟情况，例如判断 目录是否存在，代码如下：

```
test -d /usr
echo $?
test -d /usr888
echo $?
```

![configure-syntax-1-3](configure-syntax\configure-syntax-1-3.png)

`$?` 变量代表上一条命令的返回状态。

如上，如果目录不存在，就会返回 非 0。

configure 里面使用 test 的地方是 `test -t 1`，我用 --help 打印不了 test 命令的文档，但是可以用 man 命令来查看 test 文档，如下：

```
man test > t.txt
```

![configure-syntax-1-4](configure-syntax\configure-syntax-1-4.png)

因此，`-t` 就是判断描述符有没打开。

------

5，`: ${ncols:=72}`  一开始就加 : 代表不要把结果作为命令来执行，示例代码如下：

![configure-syntax-1-5](configure-syntax\configure-syntax-1-5.png)

从上图可以看出，当 ncols 变量的值为空的时候，`:=` 就会进行赋值成 72，但是由于前面没有加 `:`，所以导致结果 72 会被作为一个命令执行。

后面 ncols 有值之后，就不会再赋值成 95 了，这就是 `:=` 赋值符号的作用。

------

6，`set --`  ，设置 $ 变量的值，用法如下：

```
set a b c
echo $1 $2 $3
```

![configure-syntax-1-6](configure-syntax\configure-syntax-1-6.png)

但是 configure 里面使用 set 前面加了两个 `--` ，这是什么意思呢？请看官方解释，如下：

> --  means "don't treat anything following this as an option"

翻译过来就是 不要把后面的参数作为一个选项，你再看下面一个例子就知道这句话什么意思了。如下：

通过 `set --help` ，可以查看到 set 命令支持的选项，里面有个 `-a` 选项，如果我们正好想把 $1 设置成 `-a` 这个字符串，下面这样是不行的。

```
set -a
echo $1
```

必须写成这样，前面加上 `--` 字符。

```
set -- -a 
echo $1
```

运行结果如下：

![configure-syntax-1-7](configure-syntax\configure-syntax-1-7.png)

------

7，`set --` 保存变量之后再恢复。在 configure 里面 有 几处地方是 `set --` 把 变量内容保存进去 `$1`，`$2` ，然后执行一个函数，执行完函数之后再恢复过来。如下：

```
 set -- $cfg "$dep_all" "$dep_any" "$dep_con" "$dep_sel" "$dep_sgs" "$dep_ifa" "$dep_ifn"
 check_deps $dep_all $dep_any $dep_con $dep_sel $dep_sgs $dep_ifa $dep_ifn
 cfg=$1; dep_all=$2; dep_any=$3; dep_con=$4; dep_sel=$5 dep_sgs=$6; dep_ifa=$7; dep_ifn=$8
```

**这么搞是因为 shell 每一个地方的变量都是全局变量**，这个 跟 C 语言不太一样，C 语言是模块化，外部的变量必须传参进去给函数，函数才能用这个变量。要不函数内部只能自己声明这个变量，当然，全局变量例外。

我用一个例子 来演示 ，示例代码如下：

```
#!/bin/sh
cfg="666"
check_deps(){
  cfg="777"
}
echo "cfg="$cfg
set -- $cfg
check_deps
echo "cfg="$cfg
cfg=$1;
echo "cfg="$cfg
```

运行结果如下：

![configure-syntax-1-8](configure-syntax\configure-syntax-1-8.png)

这是因为 shell 没有局部变量这一说，任何一个地方都是全局变量，所以他要先保存，然后恢复。因为函数内部可能修改了变量的值。

------

8，`-n` ，这个语法是判断字符串长度是否为 0 ，有时候前面会省略 test 命令。

------

