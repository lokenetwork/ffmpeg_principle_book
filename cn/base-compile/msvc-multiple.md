# MSVC编译多个C程序文件—编译链接基础知识

<div id="meta-description---">MSVC编译多个C程序文件</div>

msvc 编译多个文件 跟 之前的 [《Linux环境编译多个C程序文件》](https://ffmpeg.xianwaizhiyin.net/base-compile/linux-c-multiple.html)类似的，编译阶段都只是处理单个文件，只有在链接阶段才是处理多个文件。

还是以 universe 项目为例，代码下载地址在之前文章。下载之后放到 D盘下，如图：

![msvc-multiple-1-1](msvc-multiple\msvc-multiple-1-1.png)

直接运行以下命令编译即可：

```
cl.exe /c earth.c
cl.exe /c moon.c
cl.exe /c sun.c
cl.exe /c zeus.c
```

也可以简写，如下：

```
cl.exe /c earth.c moon.c sun.c zeus.c
```

提示：不指定 `/Fo` 默认就取输入文件名。

上面两种编译方式是一样，都只是进行单文件编译，不会进行链接操作。



------

接下来执行链接操作。如下：

```
link.exe /DEBUG /OUT:zeus.exe earth.obj moon.obj sun.obj zeus.obj
```

![msvc-multiple-1-2](msvc-multiple\msvc-multiple-1-2.png)

------

现在有一个问题，如果 编译 moon.c 的时候指定 /MD 动态链接到 C运行时，其他都是默认的 /MT，会有什么问题呢？如下：

```
cl.exe /c earth.c
cl.exe /MD /c moon.c 
cl.exe /c sun.c
cl.exe /c zeus.c
link.exe /DEBUG /OUT:zeus.exe earth.obj moon.obj sun.obj zeus.obj
```

![msvc-multiple-1-3](msvc-multiple\msvc-multiple-1-3.png)

如上图，会冲突，所以必须统一用 MT 或者统一用 MD。
