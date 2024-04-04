# Qt安装教程

Qt 实际上是一个 C++ 的桌面图形窗口类库，就是一些 lib 跟 dll ，只要把这些 类库引入自己的项目，就能调qt类库做一些操作，例如创建窗口。

所以你可以把 这些 qt 的 类库配置在 vs2019， clion 里面使用都是可以的。Qt 公司还专门给 Visual Studio 提供了一个 Qt 插件。

Visual Studio Qt 插件 是“ Visual Studio Add- in 2. 0. 0 for Qt5 MSVC 2015”， 可以从Qt官网下载并安装。

不过 qt 官方提供了一个集成开发工具，就是 qt creator。这个 集成开发工具 也比较好用

------

下面开始介绍 Qt 环境的安装过程，

Qt 5.15 之后已经不提供离线安装包了，就是那个 3.7G 的 exe 安装包。请看[官方说明](https://download.qt.io/archive/qt/5.15/5.15.2/OFFLINE_README.txt)，所以只能用在线安装包。



------

1，下载在线安装包 [QT 在线安装包链接](https://download.qt.io/archive/online_installers/4.2/qt-unified-windows-x86-4.2.0-online.exe)，然后用以下命令启动安装包，**切换成中科大的源**，一定要切换源，要不下载很慢。

**提示：只有命令行启动才能指定 源 参数，直接点击 exe 无法指定参数。**

```
.\qt-unified-windows-x86-4.2.0-online.exe --mirror https://mirrors.ustc.edu.cn/qtproject
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-0-1.png)

2，指定qt 安装路径，我放在 C 盘里面，如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-1-3.png)

3，选择安装组件，在这一步可以选择各种版本的 qt 以及相关的配套软件。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-1-3-1.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-1-3-3.png)

------

然后继续点下一步，下一步，等待安装完毕即可

重点提示：安装完毕之后，如果发现有些组件漏了没勾，执行以下命令可以重新勾选。

**注意一定要用命令行打开，这样才能切换源，要不会拉取失败，只显示已安装的组件，没有其他组件给你勾。**

```
C:\Qt\MaintenanceTool.exe --mirror https://mirrors.ustc.edu.cn/qtproject
```

------

下面讲一下 qt 各种组件之间的关系。

上面 第三步 勾选的那些组件，`Qt 6.1.3`，`Qt 6.0.4` ，`Qt 5.15.2` ，这些其实对应 qt creator 里面的qt version。然后 MinGW 8.1.0 32bit 是编译器。Qt 6.1.3，Qt 6.0.4 这些我理解为Qt 框架的版本的。这些组件在qt creator里面的关系如下图：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-1-3-4.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/01/FFmpeg-1-3-5.png)

上图中，Qt 目录有 msvc 的编译器。vs2019 也有自己的 msvc 编译器的。他们可能版本不一样，但都是 msvc 。只是各自用各自的。

------

因此，qt creator 的集成开发环境是支持 两种编译器的，MinGW 跟 MSVC。这两种编译器有什么不同，推荐阅读以下文章。

1，[《MSVC编译环境介绍》](https://ffmpeg.xianwaizhiyin.net/base-compile/msvc.html)

2，[《MinGW介绍》](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-intro.html)

3，[《MinGW的优势》](https://ffmpeg.xianwaizhiyin.net/base-compile/mingw-good.html)
