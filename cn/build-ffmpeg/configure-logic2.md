# configure逻辑分析-B章—FFFmpeg configure源码分析

从 4020 行开始，就是正式解析 `configure` 参数的时候了，如下：

![configure-logic2-1-1](configure-logic2\configure-logic2-1-1.png)

解析 `configure` 参数的代码，不需要特别分析，就是执行一些逻辑，设置一些变量为 yes。

------

4130 ~ 4134 行代码，是把 环境变量 export 一下，为什么要这么搞，我也不太清楚，埋个坑，后面分析。

![configure-logic2-1-2](configure-logic2\configure-logic2-1-2.png)

------

4133 ~ 4212 行，都是调之前定义的函数来做一些逻辑，没有特别需要讲解的地方。

------

从 4214 行 开始，就开始往 日志文件里面写数据了，如下，先保存 configure 的参数。

```
echo "# $0 $FFMPEG_CONFIGURATION" > $logfile
set >> $logfile
```

`set >>` 是什么意思呢？我暂时也没明白，埋个坑，后面补充。



------

4225 ~ 4344 行，是处理 tool_chain （编译工具链）的逻辑的，FFmpeg 能在各种环境下编译，都是因为这些代码。

![configure-logic2-1-3](configure-logic2\configure-logic2-1-3.png)

------

4348 ~ 4360 是处理 `cuda` 的逻辑代码，如下：

![configure-logic2-1-4](configure-logic2\configure-logic2-1-4.png)

------

4367 ~ 4375，是检测 `pkg-config` 命令有没安装。

------

4381 ~ 4388，定义可执行文件的后缀，比较重要，例如 windows 的就是 exe 后缀。

![configure-logic2-1-5](configure-logic2\configure-logic2-1-5.png)

------

4390 ~ 4422 是处理临时目录跟文件的逻辑，如下：

![configure-logic2-1-6](configure-logic2\configure-logic2-1-6.png)

configure 运行的时候，会创建一个 `ffconf.xxx` 的文件夹，然后生成临时文件都放在  `ffconf.xxx` 里面。

重点是 最后两句 trap，这里是绑定退出逻辑，shell 执行完毕后悔删除  `ffconf.xxx` 文件夹，所以你很少见到这个临时文件夹，所以调试阶段可以把 trap 注释掉。

------

4424 ~ 4451 主要是创建一些临时文件，注意，最后会创建一个 test.sh 可执行文件，并且尝试执行，如下：

![configure-logic2-1-7](configure-logic2\configure-logic2-1-7.png)

------

4453 ~ 4585， 都是一些编译工具链 转换 flags 的代码，在 《configure函数分析-C章》 稍微讲过 `msvc_flags` 这个函数，其他函数也是类似的。

------

4777 ~  4847 都是编译工具链相关代码，调了两个函数 `probe_cc` 跟 `set_ccvar` ，这两个函数都比较复杂，需要结合一些编译例子来讲解，所以暂时跳过，知道他们是设置编译工具的代码就行了。

------

4849 ~ 4856，给 编译器加上 搜索路径，如下：

![configure-logic2-1-8](configure-logic2\configure-logic2-1-8.png)

------

4858 ~ 4889 是跨平台编译的，暂时不管。

------

4892 ~ 4936 是规范化 `$arch` 变量，因为指定指令集有多种写法，例如 iPad 跟 iPhone 都是都是 `arm` 架构。

![configure-logic2-1-9](configure-logic2\configure-logic2-1-9.png)

------

4938 ~ 5251，是给编译器加进去跟 CPU 相关的 flags。

![configure-logic2-2-1](configure-logic2\configure-logic2-2-1.png)

注意 `cpuflags` 这个变量，后续会用到的。如下：

![configure-logic2-2-2](configure-logic2\configure-logic2-2-2.png)

------

5253 ~ 5264 ，是实际用 编译器 编译一下代码，运行一下有没问题。

![configure-logic2-2-3](configure-logic2\configure-logic2-2-3.png)

------

5266 ~ 5291 ，都是检查设置一些 编译器的 flags

![configure-logic2-2-4](configure-logic2\configure-logic2-2-4.png)

上面的代码可以看出，FFmpeg 的 C 代码是 C99 标准的。

注意 `check_64bit` 函数，他是用 指针来判断当前环境是 64 位还是 32 位的。

------

5293 ~ 5327 是对 64 位还是 32 位 做一些设置。

![configure-logic2-2-5](configure-logic2\configure-logic2-2-5.png)

------

5329 ~ 5588，是针对不同 的操作系统做一些配置，例如每个系统的动态库都不太一样，windows 的是 dll，其他平台不是，还有一些乱七八糟的情况。

![configure-logic2-2-6](configure-logic2\configure-logic2-2-6.png)

------

5590 ~ 5690 ，是检查一个 link 软连接工具是否正常。

------

5612 ~ 5715， 是 `probe_libc` 函数的实现。主要设置一些 编译器选项，还有确定 C 语言的运行时类型 `libc_type`

![configure-function6-1-7](configure-function6\configure-function6-1-7.png)

