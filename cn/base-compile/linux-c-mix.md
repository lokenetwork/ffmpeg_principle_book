# Linux环境混合使用静态库与动态库—编译链接基础知识

<div id="meta-description---">Linux环境混合使用静态库与动态库，在实际工程项目里面，可能会遇到一些第三方提供的动态库，跟一些第四方提供的静态库</div>

目前 静态库 跟 动态库 编译，使用的方法都讲了一遍。但是在实际工程项目里面，可能会遇到一些第三方提供的动态库，跟一些第四方提供的静态库。

这些库都需要链接进去你自身的项目进行调用，本文主要讲解混合调用的方式。

还是以之前的例子为基础，libstar.so 动态库已经编译好，并且放置在 `/usr/local/star/lib` 目录下。现在来了一个新的大佬 theseus （波塞冬的儿子忒修斯）。

忒修斯 不但可以操作 3 颗星球，还会做饭。所以我们需要封装一个 libcook.a 静态库给 忒修斯 调用，完整的项目下载，[theseus](https://pan.baidu.com/s/1D-4ae5k3LRcOPr8ls8KVvA ) ，提取码：3yil 

下载完成 theseus 项目之后，请放置到 Document 项目，如下图：

![linux-c-mix-1-1](linux-c-mix\linux-c-mix-1-1.png)

开始执行以下命令开始编译：

```
cd ~/Documents/theseus/libcook
gcc -c -o chicken.o chicken.c
gcc -c -o fish.o fish.c
gcc -c -o noodle.o noodle.c
ar -rcs libcook.a noodle.o fish.o chicken.o
# 把 libcook.a 移动到 /usr/lib
sudo mv libcook.a /usr/lib
```

继续编译：

```
cd ~/Documents/theseus
gcc -c -o theseus.o theseus.c
gcc -o theseus theseus.o -Wl,-Bstatic -lcook -Wl,-Bdynamic -lstar -L/usr/local/star/lib
```

运行结果如下：

![linux-c-mix-1-2](linux-c-mix\linux-c-mix-1-2.png)

这样 `libstar` 就是动态链接，而 `libcook` 就是静态链接。

------

提示，其实很少情况会 静态库动态库混合使用，如果用静态库，大部分场景是为了兼容性，所有的库都是静态链接，包括 `libc` 库，这样生成的二进制文件就什么动态库都不依赖，很方便使用。

再说一个例子，例如 A 库依赖 B库， C 库也依赖 B库。如果 A 库是静态链接，那 B 库可以是静态链接或者动态库链接。

B 库不能同时静态 跟 动态链接进去二进制文件，会冲突。

记得，链接操作只发生在链接阶段，不是编译阶段。所以 A 库可以是静态链接，但是 B 库是动态链接。
