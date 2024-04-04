# Linux环境封装静态库成动态库—编译链接基础知识

<div id="meta-description---">Linux环境封装静态库成动态库</div>

前文[《Linux环境封装静态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-static2.html)是把往一个静态库加一些内容，封装成新的静态库，实际上就是解开 .a 还原成 .o 文件，再重新打包。

但是可能会有这么一种情况， 我们需要 把 libstar.a 加上控制生死的代码，封装成 libpower.so 动态库给 hades （冥王）使用。

其实也可以 把 libstar.a 解开还原成 多个 .o 文件，再一起编译出动态库。

那可不可以不解开，直接使用 ibstart.a，如下：

```
gcc -fPIC -shared -o libpower.so libstar.a dog.o pig.o 
ldd -r libpower.so
gcc -o hades hades.o libpower.so
```

![linux-c-shared2-1-1](linux-c-shared2\linux-c-shared2-1-1.png)

从上图可以看出，虽然可以正常生成动态库，但是链接到 hades 的时候直接报错了。为什么会报错，我也不知道，埋个坑，后面填。

因此，还是需要 解开还原成 多个 .o 文件，再一起编译出动态库，如下：

```
ar -x libstar.a
gcc -fPIC -shared -o libpower.so dog.o pig.o moon.o earth.o sun.o
gcc -o hades hades.o libpower.so
```

![linux-c-shared2-1-2](linux-c-shared2\linux-c-shared2-1-2.png)
