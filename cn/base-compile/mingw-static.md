# MinGW编译静态库—编译链接基础知识

以之前的  [universe](https://pan.baidu.com/s/1_reA14rpTZ6PTwY7Qc_q2Q) 项目为例，提取码：mku9 。请下载后放到 `C:\MinGW\projects` 目录，如下：

![mingw-static-1-1](mingw-static\mingw-static-1-1.png)

现在用 MinGW 的 gcc 来编译出 libstar.a 静态库给 zeus 使用，如下：

```
cd C:\MinGW\bin
.\gcc.exe -c -o C:\MinGW\projects\universe\earth.o C:\MinGW\projects\universe\earth.c
.\gcc.exe -c -o C:\MinGW\projects\universe\sun.o C:\MinGW\projects\universe\sun.c
.\gcc.exe -c -o C:\MinGW\projects\universe\moon.o C:\MinGW\projects\universe\moon.c
.\ar.exe -rcs C:\MinGW\projects\universe\libstar.a C:\MinGW\projects\universe\moon.o C:\MinGW\projects\universe\sun.o C:\MinGW\projects\universe\earth.o
```

上面这些命令，参数跟 在 [《Linux环境编译静态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-static.html)，是一样的。都是用 ar 命令来打包。

![mingw-static-1-2](mingw-static\mingw-static-1-2.png)

现在用 gcc 来使用这个 静态库，如下：

```
.\gcc.exe -c -o C:\MinGW\projects\universe\zeus.o C:\MinGW\projects\universe\zeus.c
.\gcc.exe -o C:\MinGW\projects\universe\zeus.exe C:\MinGW\projects\universe\zeus.o C:\MinGW\projects\universe\libstar.a
```

![mingw-static-1-3](mingw-static\mingw-static-1-3.png)

------

扩展知识： MinGW 的gcc 编译出来的 libstar.a 静态库能不能给 MSVC 使用，试一下：

```
cd C:\MinGW\projects\universe\
cl.exe /c zeus.c
link.exe /OUT:zeus.exe zeus.obj libstar.a
```

![mingw-static-1-4](mingw-static\mingw-static-1-4.png)

这个其实也是跟 交叉使用 编译器 链接器一样的。因为 libstar.a 就是多个 .o 文件打包在一起而已。
