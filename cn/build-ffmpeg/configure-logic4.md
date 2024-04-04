# configure逻辑分析-终章—FFFmpeg configure源码分析

6946 ~ 7072，是对各种编译器做处理，检查各种 flags，通过就加进去 给编译器使用。

![configure-logic4-1-1](configure-logic4\configure-logic4-1-1.png)

------

7100 ~ 7104 行就正式开始调 那个 `check_deps` 函数了，如下：

![configure-logic4-1-2](configure-logic4\configure-logic4-1-2.png)

`check_deps` 就是检查 前面那些代码设置的 变量 yes no 是否有问题，例如 A 依赖 B，A 是yes ，那 B 就不能是no。

------

7120 ~ 7166，有两个函数 `flatten_extralibs`，`flatten_extralibs_wrapper` ，这些我没怎么细看，暂时不管。

------

7194 ~ 7198，代码如下：

```
# Check if requested libraries were found.
for lib in $AUTODETECT_LIBS; do
    requested $lib && ! enabled $lib && die "ERROR: $lib requested but not found";
done
```

`AUTODETECT_LIBS` 里面有一些库是必须的，如果 是 no ，就会退出。

------

7198 ~ 7284 的都是逻辑代码，跳过，不讲解。



------

到 7284 ，configure 这个 shell 脚本的 环境检测，库检测 等等代码，可以说已经执行完毕。下面就是打印结果到控制台，如下：

![configure-logic4-1-3](configure-logic4\configure-logic4-1-3.png)

`configure` 成功之后，往控制台输出信息，就是这段代码干的。输出信息代码 一直 到 7432 行。

------

到 7432 行，打印完信息之后，还需要把结果 转存到 相关的 .h 文件，.mak 文件，.sh 文件。如下：

![configure-logic4-1-4](configure-logic4\configure-logic4-1-4.png)

最后一句代码是

```
cp_if_changed $TMPH ffbuild/config.sh
```

`$TMPH` 其实是给 `.h` 头文件用的，他临时用了一下，给 `.sh` 文件用。因为没有定义 `$TMPSH`  这个变量用

------

最后，说下 `configure` 脚本创建了哪些文件，内容是什么。`configure` 创建的文件都在 `ffbuild` 目录。

**1，config.log**，这个是日志文件，记录了 `configure` 整个 shell 脚本的执行过程，编译了哪些代码都会记录。

**2，config.sh**，这是一个 shell 脚本，给 `pkg-config` 使用的。安装 `FFmpeg` 动态库到系统目录的时候会用到。

**3，config.mak**，这是一个 `makefile` 格式的文件，之前说过 `configure` 的 8千行代码，就是为了往编译器，链接器传递参数，如下：

![configure-logic4-1-5](configure-logic4\configure-logic4-1-5.png)

从上图可以看出，我用的 是 `msvc` 的 `cl.exe` 编译器，编译器选项是 CFLAGS

------

`ffbuild` 目录还有一些本来就存在的文件，我也讲解一下。

**1，arch.mak** ，跟指令集相关。

**2，common.mak**，common bits used by all libraries

**3，library.mak** ，这个应该是 library 相关的 makefile。

**4，libversion.sh**

**5，pkgconfig_generate.sh**

**6，version.sh**，打印 编译好的 FFmpeg 版本的 shell

