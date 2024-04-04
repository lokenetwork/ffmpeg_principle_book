# configure逻辑分析-A章—FFFmpeg configure源码分析

大概在 2532 行之前，configure 都是在 定义 一些通用的函数，变量。2532 行之后 configure 脚本开始执行一些具体逻辑。

本章内容就开始 分析 configure 脚本的逻辑，如下：

![configure-logic-1-1](configure-logic\configure-logic-1-1.png)

上图中，都是在定义依赖，主要是 `check_deps` 函数在用这些依赖变量，推荐看 [《configure函数分析-B章》](https://ffmpeg.xianwaizhiyin.net/build-ffmpeg/configure-function2.html)，了，里面有讲 `check_deps` 函数的内部逻辑。

从 2533 ~ 2594 行，都是在定义依赖，所以可以简单跳过，只要了解  `check_deps` 函数的逻辑即可。

------

下图是 2597 行的代码，要仔细讲一下。

![configure-logic-1-2](configure-logic\configure-logic-1-2.png)

`filter_out` 函数就是排除 数组里面的 mmx 选项。



------

2603 ~ 3749 行 都是定义一些 依赖，deps，select，suggest， if ， if_any  之类的，还是那一句，可以跳过，只要了解  `check_deps` 函数的逻辑即可。

不过由于 这些 依赖代码，有近 x 千，比较重要，我还是 重复 讲一下他这些依赖是怎么被   `check_deps` 函数 使用的。

![configure-logic-1-3](configure-logic\configure-logic-1-3.png)

从上图可以看到， `check_deps` 函数会检测 一个 cfg 是否通过，如果通过，就会 deep 来设置相关的 select ，suggest  成 yes 或者 no。

------

3751 ~ 3795 行是设置一些 默认的变量，包括 path （路径），toolchain（编译工具链），os （当前操作系统），machine （指令集），

如下：

![configure-logic-1-4](configure-logic\configure-logic-1-4.png)

------

3797 ~ 3823 行，是设置一些默认的变量 为 yes， **注意这里不一定是最后的值**，因为还没开始解析 configure 的参数。configure 的参数可能会再次修改这些变量

而且也没开始检测，如果检测不通过还会设置为 no。

![configure-logic-1-5](configure-logic\configure-logic-1-5.png)

------

3825 ~ 3870 行主要是 build settings，如下：

![configure-logic-1-6](configure-logic\configure-logic-1-6.png)

注意，这些值也不一定就是最终的值，例如这个 `SHFLAGS`， 还可能会在 5300 行的位置被改变。

![configure-logic-1-7](configure-logic\configure-logic-1-7.png)

------

```
# since the object filename is not given with the -MM flag, the compiler
# is only able to print the basename, and we must add the path ourselves
DEPCMD='$(DEP$(1)) $(DEP$(1)FLAGS) $($(1)DEP_FLAGS) $< 2>/dev/null | sed -e "/^\#.*/d" -e "s,^[[:space:]]*$(@F),$(@D)/$(@F)," > $(@:.o=.d)'
DEPFLAGS='-MM'
```

上面这几句代码，我也没看懂是在做什么，后面再补充。

------

![configure-logic-1-8](configure-logic\configure-logic-1-8.png)

上图创建了一个 `ffbuild` 目录，然后设置 `source_path` 变量。

------

![configure-logic-1-9](configure-logic\configure-logic-1-9.png)

把 configure 的参数格式化后 保存进去 `FFMPEG_CONFIGURATION` 变量

------

3900 ~ 3965 行代码，都是在定义一些列表，如下：

![configure-logic-2-1](configure-logic\configure-logic-2-1.png)

`find_things_extern` 函数在  [《configure函数分析-E章》]( https://ffmpeg.xianwaizhiyin.net/build-ffmpeg/configure-function6.html) 讲解过了。最后就是 enable 这些列表的变量。设置为 yes， 如下：

![configure-logic-2-2](configure-logic\configure-logic-2-2.png)

注意，这里也不是最终值，因为还未开始 解析 configure 的参数。

