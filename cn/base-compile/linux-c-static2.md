# Linux环境封装静态库—编译链接基础知识

<div id="meta-description---">先解释 一下 编译静态库 跟 封装静态库的 区别，编译静态库 就是前文中的 把多个 .o 文件打包成一个 libxxx.a 静态库。而 封装静态库 是已经有一个第三方的 静态库，我们想在 这个第三方静态库 上面再加一些功能，提供给客户使用。</div>

先解释 一下 编译静态库 跟 封装静态库的 区别，编译静态库 就是前文中的 把多个 .o 文件打包成一个 libxxx.a 静态库。

而 封装静态库 是已经有一个第三方的 静态库，我们想在 这个第三方静态库 上面再加一些功能，提供给客户使用。

还是以之前的 poseidon （波塞冬）项目为基础。现在又来了一位新大佬，hades （冥王），这位大佬除了 可以操作 3颗 星球之外，还可以控制世间万物的生死。

所以我们需要基于 libstar.a ，再加一些功能给 hades 用。相关文件我已经创建好了，下载 [hades](https://pan.baidu.com/s/1OoNtw_Y0vZyAVOA18I-eJw ) 即可，提取码：wfny 。



------

把 hades  放在 Document 目录下，如下图：

![linux-c-static2-1-1](linux-c-static2\linux-c-static2-1-1.png)

我新建了 两个 文件 dog.c 跟 pig.c ，冥王可以控制 猪 跟 狗死亡。

现在开始封装 新的静态库 libpower.a ，命令如下：

```
gcc -c -o dog.o dog.c
gcc -c -o pig.o pig.c
# 解压原来的静态库
ar -x libstar.a
ar -rcs libpower.a moon.o sun.o earth.o dog.o pig.o
```

![linux-c-static2-1-2](linux-c-static2\linux-c-static2-1-2.png)

编译新的静态库，需要先解压。用 objdump 查看一下 libpower.a 的内容：

```
objdump -d libpower.a > libpower.txt
```

![linux-c-static2-1-3](linux-c-static2\linux-c-static2-1-3.png)

实际上，就是解压之后，再打包，我们使用第三方项目的静态库的时候也可以这样，先解压，再打包我们自己的 .o 文件进去。

------

libpower.a 这个静态库的使用方法，也是跟之前类似的，可以把它放到 `/usr/lib` 下面，然后用以下命令编译：

```
sudo mv libpower.a /usr/lib
gcc -c -o hades.o hades.c
gcc -o hades hades.o -static -lpower
```

![linux-c-static2-1-4](linux-c-static2\linux-c-static2-1-4.png)

至此，封装第三方静态库的方法讲解完毕，实际上就是解压，然后再打包压缩。

如果你不解压，直接把 libstar.a 跟其他两个 .o 文件一起打包，会有问题。

