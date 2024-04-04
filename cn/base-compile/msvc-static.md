# MSVC编译静态库—编译链接基础知识

<div id="meta-description---">MSVC编译静态库</div>

在 Windows 系统里面，静态库的正式名称是 object library（对象库）。

参考 [《Linux环境编译静态库》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-static.html)，要编译出一个 静态库给 zeus.c （宙斯）用。项目代码还是 D 盘的 universe。

先执行以下命令生成 obj 目标文件：

```
cl.exe /c earth.c moon.c sun.c 
```

Linux 下生成 静态库使用的是 ar 打包命令，而 Windows 下使用的是 [lib.exe](https://docs.microsoft.com/zh-cn/cpp/build/reference/lib-reference?view=msvc-170) 命令，如下：

```
lib.exe /OUT:star_static.lib earth.obj moon.obj sun.obj
```

![msvc-static-1-1](msvc-static\msvc-static-1-1.png)



------

再用 PEView 来查看一下 star_static.lib 的内容，如下：

![msvc-static-1-2](msvc-static\msvc-static-1-2.png)

可以看到，在 Windows 下生成静态库也是一个 归档操作。

现在来反编译看一下 star_static.lib 的汇编代码，如下：

```
dumpbin /DISASM star_static.lib
```

![msvc-static-1-3](msvc-static\msvc-static-1-3.png)

可以看出来跟 Linux 一样，也是没有修正地址。

------

下面演示一下如何使用静态库，命令如下：

```
cl.exe /c zeus.c
link.exe /DEBUG /OUT:zeus.exe zeus.obj star_static.lib
```

提示：如果是正式发布，可以把 `/DEBUG` 换成 `/REALSE`

![msvc-static-1-4](msvc-static\msvc-static-1-4.png)

扩展知识：

1，由于静态库跟动态库的导入库都是 .lib 后缀，他们不能同名。因此必须有一种方法区分 静态库 跟 导入库。主流有两种命名方式。

- cuda 规范，用 **库名_static.lib** 作为静态库 ，用 **库名.lib **作为动态库的导入库。所以上面的 star 静态库按 cuda 的规则就是 star_static.lib 。
- 普通 规范，用 lib 库名.lib 作为静态库，静态库前面有个 lib 前缀，导入库没有 lib 前缀。所以上面的 star 静态库按 普通的规则就是 libstatic.lib 

2，link.exe 没有类似 Linux gcc 的 `-l` 选项的功能，要引入库，只能写全称，不过可以通过 `/LIBPATH` 指定最优先的搜索路径。

------

上面是直接使用 cl.exe 跟 link.exe 来生成跟使用静态库，要敲很多命令。Windows 平台是以界面化操作出名了，所以 vs2019 有更方便的创建跟使用静态库的方式。

先使用 vs2019 创建一个空项目 star_static，如下：

![msvc-static-1-5](msvc-static\msvc-static-1-5.png)

![msvc-static-1-6](msvc-static\msvc-static-1-6.png)

然后再打开项目配置，设置 Platform 跟 General ➜ Configuration Type，如下：

![msvc-static-1-7](msvc-static\msvc-static-1-7.png)

然后把 之前的 sun.c , moon.c ，earth.c 赋值到 项目下，如下：

![msvc-static-1-8](msvc-static\msvc-static-1-8.png)

虽然 文件已经 复制到项目目录了，但是还没有加进去 solution里面的，就是那个 star_static.vcxproj.filters xml 配置文件。需要点击  Add ➜ Exiting Item 添加进去解决方案。如下：

![msvc-static-1-10](msvc-static\msvc-static-1-10.png)

添加进去之后，再点击 Save ALL，star_static.vcxproj.filters 的内容就会发生改变，如下：

![msvc-static-1-11](msvc-static\msvc-static-1-11.png)

因此，vs2019 界面上执行 Add ➜  Exiting Item 实际上就是修改 star_static.vcxproj.filters 文件，你手动改这个文件，然后再重新加载项目，效果是一样的。

说白了，vs2019 就是通过界面管理这些 xml 文件，一个 项目有好几个 xml 文件。刚开始设置 Configuration Type 也是改 xml 文件的内容。

------

添加完文件之后，点击菜单栏的 Build ➜ Rebuild Solution，执行之后如下，会发现没有生成 star_static.lib 静态库：

![msvc-static-1-12](msvc-static\msvc-static-1-12.png)

因此要排查一下 vs2019 的整个运行过程，生成静态库有两个阶段，第一阶段是用 cl.exe 编译出来 obj文件，这一步没有错。

第二步是 调 lib.exe 生成静态库，这一步没有执行成功，所以我们把 vs2019 的 MSbuild 日志开到最大，这样就可以看到详细的日志。

提示：点击 菜单栏的 Tool ➔ Options，即可弹出以下配置窗口。 箭头 ➔

![msvc-static-1-13](msvc-static\msvc-static-1-13.png)

![msvc-static-1-14](msvc-static\msvc-static-1-14.png)

从日志上可以看到 编译 sun.c 使用的选项，比我们手动敲命令多了很多选项，这是 vs2019 默认的一些选项，不用管，vs2019 也是使用 cl.exe 来编译的。

在 vs2019 看这些日志不太方便，所以我复制日志到 clion 来看，如下，直接搜索 sun.obj 

![msvc-static-1-15](msvc-static\msvc-static-1-15.png)

可以看到，生成静态库的路径是 `C:\Users\loken\source\repos\star_static\x64\Debug\star_static.lib`。用 dir 查看这个文件，确实存在。

![msvc-static-1-16](msvc-static\msvc-static-1-16.png)

**因此，之前是我看错目录了，所以没找到 star_static.lib 。**

------

通过这次排查，我们发现 实际上 vs2019 界面上这么多纷繁复杂的功能，无非就是干3件事情。

1，管理 xml 文件。很多编译选项都是通过 xml 属性传递给 cl.exe 的。

2，调 cl.exe 编译出来 obj 目标文件。

3，调 lib.exe 生成 静态库。

在现在这个时代，我不建议大家手写 cl.exe 等 命令来配置项目的编译过程，最好使用 vs2019 来配置，因为有很多默认选项你不知道。

学习 cl.exe ，lib.exe ，link.exe 的目的在于，知道 vs2019 编译一个项目的内部过程，如果出错了，可以通过日志 查看 cl.exe 的各个选项去反推到底是哪里出问题。而不是乱点界面上的配置跟按钮。

------

现在来验证一下 vs2019 的 star_static.lib 好不好用，复制到 D 盘的 universe 项目，再编译一次，如下：

```
cl.exe /MDd /c zeus.c
link.exe /OUT:zeus.exe zeus.obj star_static.lib
```

上面的 /MDd 代表使用 debug 版本的动态链接，因为 star_static.lib 也是使用 /MDd 生成的，要保持C运行时库一致。

![msvc-static-1-17](msvc-static\msvc-static-1-17.png)

如上图所示，vs2019 生成的 star_static.lib 可以正常使用，只是链接的时候报了个 waring ，估计是某些选项 vs2019 设置不太对，不过不碍事。

参考资料：

1，[《lib.exe使用手册》](https://docs.microsoft.com/zh-cn/cpp/build/reference/lib-reference?view=msvc-170)

2，[《VS2019静态库的创建》](https://blog.csdn.net/SimplexXx0/article/details/122018659)

