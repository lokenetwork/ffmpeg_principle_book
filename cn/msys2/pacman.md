# pacman包管理器介绍

<div id="meta-description---">msys2 的包管理器是 pacman ，pacman 原来是 Arch Linux 平台的管理器。msys2 参考 Arch Linux 实现的 pacman。</div>

msys2 的包管理器是 pacman ，pacman 原来是 Arch Linux 平台的管理器。msys2 参考 Arch Linux 实现的 pacman。

首先介绍一个 pacman 命令的用法，可以使用 --help 查看支持的选项，如下：

![pacman-1-1](pacman\pacman-1-1.png)

这样看起来，帮助文档好像很少，但其实不是的，如果你想看 -S 下面还有哪些选项，可以使用以下命令：

```
pacman -S --help
```

![pacman-1-2](pacman\pacman-1-2.png)

**重点：任何 Linux 的软件都可以用 --help 找到一些关键的使用资料，而且大多数的 --help 是支持子文档查询，例如上面加了 -S， 只看这个 -S 的相关文档。**



------

1，mingw32，MinGW 的32位环境的软件

2，mingw64，MinGW的64位环境的软件

3，ucrt64，微软的通用运行时环境，u 代表 universe（通用）。

4，clang32，clang 32位环境，clang 是一个编译器。

5，clang64，clang 64位环境

6，clangarm64，arm架构的CPU 的 clang 环境。

7，msys

你可以把上面的 7个 东西都理解成一个 子系统。mingw32 跟 mingw64 是比较常用的子系统。

在 [msys2 repository](https://packages.msys2.org/queue)  可以看到 mingw32 跟 mingw64 环境有哪些软件可以安装，如下：

![intro-1-4](intro\intro-1-4.png)

![intro-1-5](intro\intro-1-5.png)

------

上图两个 SDL2 的软件名如下：

```
mingw-w64-i686-SDL2
mingw-w64-x86_64-SDL2
```

i686 代表是 32位的程序，x86_64 代表是 64位的程序。不用管前面的 w64 是啥，其实 mingw-w64 是一个整体，把他理解成一个名字即可，不要看到 64 以为是 64位的，例如为什么**筷子** 不是竹子做的，而是不锈钢做的，明明筷这个字是竹字头的。所以纠结这个 mingw-w64 代表什么没有意义，这个就是一个名字。

------

使用pacman而不是自己编译和安装程序有很多好处：

- 轻松升级：*pacman* 会在更新可用时立即更新已安装的软件包
- 依赖检查：*pacman* 会为你处理依赖问题，只需要指明程序（名），*pacman* 就会将它和它所需的所有其他程序都一起安装。
- 干净卸载：*pacman* 持有软件包包含的所有文件的列表。这样一来，当你决定移除软件包时，不会无意留下任何文件。

------

pacman 比较常用的命令如下：

###### 1，查询 云端仓库是否有某个软件

```
pacman -Ss sdl
```

`-Ss` 后面是 正则表达式字符串。

###### 2，查看云端软件的具体信息

```
pacman -Si mingw-w64-x86_64-SDL2
```

###### 2，安装一个软件

```
pacman -S mingw-w64-x86_64-SDL2
```

3，查询一个软件是否安装

```
pacman -Qs mingw-w64-x86_64-SDL2
```

3，查询本地已安装软件的具体信息

```
pacman -Qi mingw-w64-x86_64-SDL2
```

4，查询本地已安装软件包所包含文件的列表

```
pacman -Ql mingw-w64-x86_64-SDL2
```

4，删除一个软件

```
pacman -R mingw-w64-x86_64-SDL2
```

------

由于 packman 安装一个软件的过程就是下载 已经编译好的 静态库，动态库或者 exe，跟一些配置相关的文件，放在指定的位置。

我们现在就来看一下， 上面 pacman 安装 SDL2 的时候 从网络上下载了什么东西，安装命令如下：

```
pacman -S mingw-w64-x86_64-SDL2
```

![pacman-1-3](pacman\pacman-1-3.png)

pacman 将下载的软件包保存在 `/var/cache/pacman/pkg/` 目录，如下：

![pacman-1-4](pacman\pacman-1-4.png)

这是一个 zst 的压缩包，我们可以用以下命令解压看一下里面的内容，如下：

```
tar -I zstd -xvf mingw-w64-x86_64-SDL2-2.0.20-1-any.pkg.tar.zst
```

![pacman-1-5](pacman\pacman-1-5.png)

![pacman-1-5](pacman\pacman-1-6.png)

可以看出来，解压后的目录结构，跟 msys2 的目录是一样的，**上图中那些 dll.a 实际上就是动态库的导入库，也就是 lib 导入库。而 .a 后缀的就是 lib 静态库。**

我们可以用以下命令查看一下 SDL 安装的位置如下：

```
pacman -Ql mingw-w64-x86_64-SDL2
```

![pacman-1-7](pacman\pacman-1-7.png)

如果执行  `pacman -R mingw-w64-x86_64-SDL2` ，上图中的文件就会被删除。

------

pacman 还有一个 命令比较常用，如果不小心删除了SDL 的 SDL2.dll 文件，怎么知道哪个文件不见了，可以用以下命令。

```
pacman -Qk mingw-w64-x86_64-SDL2
```

![pacman-1-8](pacman\pacman-1-8.png)

这个查出缺少文件的功能，应该是通过对比本地数据库，或者跟云端的数据对比得出来的。

------

pacman 通常会自动帮我们处理软件的依赖，但如果想看一下 软件的依赖，可以使用以下命令。

```
pactree mingw-w64-x86_64-SDL2
```

------

上面都是下载 MinGW64 位的软件，其实只要换个名字，就能下载 32位 的SDL程序，如下：

```
pacman -S mingw-w64-i686-SDL2
pacman -Ql mingw-w64-i686-SDL2
```

mingw32 位的程序会放在 根目录的 /minw32 目录，如下：

![pacman-1-9](pacman\pacman-1-9.png)



参考资料：

1，[《pacman》](https://wiki.archlinux.org/title/Pacman_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%BD%AF%E4%BB%B6%E4%BB%93%E5%BA%93) - 维基百科

