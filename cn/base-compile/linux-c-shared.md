# Linux环境编译动态库—编译链接基础知识

<div id="meta-description---">Linux环境编译动态库</div>

我们之前已经 编译过 libstar.a 静态库 给 zeus（宙斯）跟 poseidon（波塞冬）使用。由于 静态库 会把内容拷贝进去程序里面，所以会加大磁盘存储空间。

如果有100个软件都用到 某个库，如果这个库是静态链接到100个软件，数据量就会很大，所以操作系统一些底层库，都是以动态库的方式提供给上层程序调用的。

而且动态库还有一个好处，可以很方便单独更新某个动态库，可以利用动态库的机制做一个插件系统。

我们如何编译出 libstar.so 动态库呢？这个就是本文的重点。

相关文件 sun.c , moon.c 跟 earth.c 还是之前在  [universe](https://pan.baidu.com/s/1_reA14rpTZ6PTwY7Qc_q2Q) 目录，提取码：mku9 。用以下命令即可 编译出 libstar.so 动态库。

```
gcc -c -o sun.o sun.c
gcc -c -o moon.o moon.c
gcc -c -o earth.o earth.c 
gcc -fPIC -shared -o libstar.so sun.o moon.o earth.o
```

`-fPIC` 是编译选项，PIC是 Position Independent Code 的缩写，表示要生成位置无关的代码。执行结果如下：

![linux-c-shared-1-1](linux-c-shared\linux-c-shared-1-1.png)



------

现在用 objdump 查看一下这个动态库的汇编代码。

```
objdump -d libstar.so > star-so-dump.txt
```

![linux-c-shared-1-1-1](linux-c-shared\linux-c-shared-1-1-1.png)

从上图可以看到，动态库跟之前的静态库不太一样，两个 call 的地址已经被修正。这是为什么呢？不是说动态链接，只有在运行的时候才会进行链接。

为什么此时此刻，就会修正 call 的地址？

注意看 `moon_rotate@plt`  后面的 plt 全称是 Procedure Linkage Table （过程链接表），我们可以看以下 callq 590 会跳到哪里，如下：

![linux-c-shared-1-1-1-1](linux-c-shared\linux-c-shared-1-1-1-1.png)

上图中的 590 是硬盘文件中的偏移，偏移590 字节就能看到一样的二进制内容。上图中的三条汇编指令，实际根本不是我们之前定义的 moon_rate 函数的指令。

这个实际上是生成 动态库的时候，给call 00 00 00 的函数引用封装一层。通过 gdb 调试会会发现，程序是先 进入 moon_rate@plt ，然后再通过 jmpq 跳转到 真正的 moon_rate 函数。

**这个过程可以理解为， 通过 Procedure Linkage Table 表，跳转到真正的函数。**

------

我们再做一个实验，生成 libstar.so 动态库的时候，把 moon.o 删掉，不加入。看看 sun.o 里面对 moon_rate 的引用会不会被修正。

提醒：注意这个 libstar-err.so，这个漏了 moon.o 的动态库后面会用来显示一个错误的情况。

```
gcc -fPIC -shared -o libstar-err.so sun.o earth.o
```

![linux-c-shared-1-1-1-2](linux-c-shared\linux-c-shared-1-1-1-2.png)

从上图可以看到，即使没有 moon.o 依然会被修正。但是 这个 libstar-err.so 是有问题的，可以使用 ldd 加上 -r 选项进行模拟重定位函数，会发现找不到 moon_rorate 函数的实现。

![linux-c-shared-1-1-1-3](linux-c-shared\linux-c-shared-1-1-1-3.png)

------

因为 sun.o 里面调了 printf 函数，所以用 ldd 查看，可以发现 libstar.so 跟 libc.so 已经建立了链接。

![linux-c-shared-1-1-2](linux-c-shared\linux-c-shared-1-1-2.png)

------

现在把 libstar.so 拷贝到 zeus 项目，执行以下命令编译。

```
gcc -c -o zeus.o zeus.c
gcc -o zeus zeus.o libstar.so
./zeus
```

![linux-c-shared-1-2](linux-c-shared\linux-c-shared-1-2.png)

从上图可以看出，虽然可以顺利链接，生成 zeus 文件，但是运行的时候却报找不到 libstar.so 动态库，这是因为 Linux 环境默认不会从当前路径 加载动态库。而 Windows 环境会从当前路径 加载动态库。

那Linux 的加载器会从哪些目录搜索加载动态库呢？业界制定了一个 FHS （File Hierarchy Standard）标准，这个标准规定了一个系统中的系统文件应该如何存放，大部分Linux系统都遵循这个标准。

共享库的存放方式也在这个 FHS 标准里面，标准定义共享库可以放在 `/lib` , `/usr/lib` , `/usr/local/lib` 这 3个目录。所以运行加载动态库的时候，也会在这 3个目录搜索。

但是还有一个常规做法，在 `/etc/ld.so.conf.d/` 添加自定义配置。我们查看一下 `ld.so.conf ` 文件的内容，如下：

![linux-c-shared-1-3](linux-c-shared\linux-c-shared-1-3.png)

发现他是把其他的文件 include 进来的，所以我们需要创建一个自己的配置文件 `/etc/ld.so.conf.d/star.conf` ，内容如下：

```
# star default configuration
/usr/local/star/lib
```

然后创建目录 `/usr/local/star/lib` ，把 libstar.so 拷贝到  `/usr/local/star/lib` 。

此时，还需要 重新加载一下之前的 star.conf 配置，执行 命令 `sudo ldconfig`。再次 运行 zeus 就没有问题了，即使把当前目录的 libstar.so 删除也能运行，因为 运行的时候是从 `/usr/local/star/lib` 目录找动态库

![linux-c-shared-1-4](linux-c-shared\linux-c-shared-1-4.png)

上面我是直接 写 动态库全称来进行链接了，其实另一种更常用的链接动态库的语法是这样的。如下：

```
gcc -o zeus zeus.o -L/usr/local/star/lib -lstar
```

`-L` 是指定库的搜索路径。这里提醒一下，**链接 跟 运行加载 是两个不同的操作**。之前 在  `/etc/ld.so.conf.d/star.conf` 文件配置的是加载的路径。链接的时候还是需要用 `-L` 参数指定。

------

之前我们生成了一个 libstar-err.so 动态库，这个动态库是没有 moon.o 的，如果 zeus 项目使用这个动态库会怎样呢？把 libstar-err.so 拷贝到 `/usr/local/star/lib`，执行下面命令再次编译。

```
mv libstar-err.so /usr/local/star/lib
gcc -o zeus-err zeus.o -L/usr/local/star/lib -lstar-err
```

![linux-c-shared-1-4-1](linux-c-shared\linux-c-shared-1-4-1.png)

可以看到，链接器直接报错，找不到 moon_rotate 的函数实现。

------

相关阅读：

1，[《C++静态库与动态库》](https://www.cnblogs.com/skynet/p/3372855.html)- 吴秦

