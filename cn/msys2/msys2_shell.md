# msys2_shell.cmd源码分析

在 msys 的安装目录有一个 [msys2_shell.cmd](./msys2_shell/msys2_shell.cmd) ，我们可以传递 参数 `-mingw32` 或者 `-mingw64` 来进入不同的环境。

![msys2_shell-0-1](msys2_shell\msys2_shell-0-1.png)

![intro-1-0](intro\intro-1-0.png)

------

msys2_shell.cmd 的使用方法，可以用 `/?` 参数来查看，如下：

![msys2_shell-0-2](msys2_shell\msys2_shell-0-2.png)



------

本文就**逐行**来分析一下 msys2_shell.cmd 的源码逻辑，重点分析 cmd 是如何解析  `-mingw32` 跟 `-mingw64`  这两个参数的，里面干了什么事情。

**调试技巧：可以使用 echo 来输出一些变量，变量名要用 %% 包起来。exit /b 命令可以退出，不执行后面的命令。**

------

```
@echo off
setlocal EnableDelayedExpansion
```

最开始的两行代码如上，`echo off` 是关闭输出回显，因为如果不关闭，每条命令都会显示在屏幕上，如下：

![msys2_shell-1-1](msys2_shell\msys2_shell-1-1.png)

**提示：rem 是注释。**

而 `setlocal EnableDelayedExpansion` 的作用请看 《[批处理中setlocal enabledelayedexpansion的作用](https://www.cnblogs.com/ydhliphonedev/archive/2012/09/25/2702092.html)》

------

```
set "WD=%__CD__%"
if NOT EXIST "%WD%msys-2.0.dll" set "WD=%~dp0usr\bin\"
set "LOGINSHELL=bash"
set /a msys2_shiftCounter=0
```

上面的代码重点如下：

1，`%__CD__%` ，这个环境变量是当前目录。

2，判断当前目录的 msys-2.0.dll 文件是否存在，我的不存在，所以把 WD 变量重新设置了。

3，`%~dp0` 这个符号比较难懂，里面的 0 代表 cmd 文件本身，然后 p 是获取 cmd 文件的路径，而 d 是获取分区号（C盘还是D盘）。

所以 WD 就是 `C:\msys64\usr\bin\`

4，`set /a msys2_shiftCounter=0 `  ，这个 `/a` 后面可以跟表达式。windows 命令通常可以使用 `/?` 来查看帮助文档，如下：

![msys2_shell-1-2](msys2_shell\msys2_shell-1-2.png)

5，msys2_shiftCounter 这个变量的实际作用是统计命令行的参数数量，每处理一个参数，msys2_shiftCounter 就会 + 1.

------

```
rem To activate windows native symlinks uncomment next line
rem set MSYS=winsymlinks:nativestrict

rem Set debugging program for errors
rem set MSYS=error_start:%WD%../../mingw64/bin/qtcreator.exe^|-debug^|^<process-id^>

rem To export full current PATH from environment into MSYS2 use '-use-full-path' parameter
rem or uncomment next line
rem set MSYS2_PATH_TYPE=inherit
```

上面这些都是需要自己开启的配置。最主要的是 `MSYS2_PATH_TYPE` 这个变量， inherit 代表把当前窗口的环境变量导入给 msys2 的命令行。

TODO，留意一下 winsymlinks 变量。

------

```
:checkparams
rem Help option
if "x%~1" == "x-help" (
  call :printhelp "%~nx0"
  exit /b %ERRORLEVEL%
)
if "x%~1" == "x--help" (
  call :printhelp "%~nx0"
  exit /b %ERRORLEVEL%
)
if "x%~1" == "x-?" (
  call :printhelp "%~nx0"
  exit /b %ERRORLEVEL%
)
if "x%~1" == "x/?" (
  call :printhelp "%~nx0"
  exit /b %ERRORLEVEL%
)
```

上面的代码是 输出 帮助信息，**重点 是exit 的 用法，用 /b 可以不退出当前窗口**。

------

```
rem Shell types
if "x%~1" == "x-msys" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=MSYS& goto :checkparams
if "x%~1" == "x-msys2" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=MSYS& goto :checkparams
if "x%~1" == "x-mingw32" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=MINGW32& goto :checkparams
if "x%~1" == "x-mingw64" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=MINGW64& goto :checkparams
if "x%~1" == "x-ucrt64" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=UCRT64& goto :checkparams
if "x%~1" == "x-clang64" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=CLANG64& goto :checkparams
if "x%~1" == "x-clang32" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=CLANG32& goto :checkparams
if "x%~1" == "x-clangarm64" shift& set /a msys2_shiftCounter+=1& set MSYSTEM=CLANGARM64& goto :checkparams
if "x%~1" == "x-mingw" shift& set /a msys2_shiftCounter+=1& (if exist "%WD%..\..\mingw64" (set MSYSTEM=MINGW64) else (set MSYSTEM=MINGW32))& goto :checkparams
rem Console types
if "x%~1" == "x-mintty" shift& set /a msys2_shiftCounter+=1& set MSYSCON=mintty.exe& goto :checkparams
if "x%~1" == "x-conemu" shift& set /a msys2_shiftCounter+=1& set MSYSCON=conemu& goto :checkparams
if "x%~1" == "x-defterm" shift& set /a msys2_shiftCounter+=1& set MSYSCON=defterm& goto :checkparams
```

上面的代码是 设置 Shell types 跟 Console types，重点要注意 [mintty.exe](https://github.com/mintty/mintty) ，msys2 的默认窗口就是调这个 mintty.exe 打开的。

![msys2_shell-1-3](msys2_shell\msys2_shell-1-3.png)

------

```
rem Other parameters
if "x%~1" == "x-full-path" shift& set /a msys2_shiftCounter+=1& set MSYS2_PATH_TYPE=inherit& goto :checkparams
if "x%~1" == "x-use-full-path" shift& set /a msys2_shiftCounter+=1& set MSYS2_PATH_TYPE=inherit& goto :checkparams
if "x%~1" == "x-here" shift& set /a msys2_shiftCounter+=1& set CHERE_INVOKING=enabled_from_arguments& goto :checkparams
if "x%~1" == "x-where" (
  if "x%~2" == "x" (
    echo Working directory is not specified for -where parameter. 1>&2
    exit /b 2
  )
  cd /d "%~2" || (
    echo Cannot set specified working diretory "%~2". 1>&2
    exit /b 2
  )
  set CHERE_INVOKING=enabled_from_arguments

  rem Ensure parentheses in argument do not interfere with FOR IN loop below.
  set msys2_arg="%~2"
  call :substituteparens msys2_arg
  call :removequotes msys2_arg

  rem Increment msys2_shiftCounter by number of words in argument (as cmd.exe saw it).
  rem (Note that this form of FOR IN loop uses same delimiters as parameters.)
  for %%a in (!msys2_arg!) do set /a msys2_shiftCounter+=1
)& shift& shift& set /a msys2_shiftCounter+=1& goto :checkparams
if "x%~1" == "x-no-start" shift& set /a msys2_shiftCounter+=1& set MSYS2_NOSTART=yes& goto :checkparams
if "x%~1" == "x-shell" (
  if "x%~2" == "x" (
    echo Shell not specified for -shell parameter. 1>&2
    exit /b 2
  )
  set LOGINSHELL="%~2"
  call :removequotes LOGINSHELL
  
  set msys2_arg="%~2"
  call :substituteparens msys2_arg
  call :removequotes msys2_arg
  for %%a in (!msys2_arg!) do set /a msys2_shiftCounter+=1
)& shift& shift& set /a msys2_shiftCounter+=1& goto :checkparams
```

上面的代码是设置一些其他的参数，重点有以下：

1，`-full-path` 跟 `-use-full-path` 是一样的，都是设置  MSYS2_PATH_TYPE=inherit ，导入当前窗口的环境变量给 msys2 的命令行。

2，`-here`，设置 msys2 的工作目录为当前目录。

3，`-where`，指定 msys2 的工作目录。

4，`-no-start` ，`-shell` ，这两个选项我也不知道有什么用，暂时不管。



------

```
rem Collect remaining command line arguments to be passed to shell
if %msys2_shiftCounter% equ 0 set SHELL_ARGS=%* & goto cleanvars
set msys2_full_cmd=%*
for /f "tokens=%msys2_shiftCounter%,* delims=,;=	 " %%i in ("!msys2_full_cmd!") do set SHELL_ARGS=%%j

:cleanvars
set msys2_arg=
set msys2_shiftCounter=
set msys2_full_cmd=

```

上面这段代码特别重要，里面有个 for 循环，主要是把一些 不是 自己解析的参数 放到 `SHELL_ARGS` 变量，传递下去。

`.\msys2_shell.cmd -mingw32 -a -b -c` ，我乱写了几个参数，断点打印一下 SHELL_ARGS 的内容，如下：

![msys2_shell-1-4](msys2_shell\msys2_shell-1-4.png)

------

```
rem Setup proper title and icon
if "%MSYSTEM%" == "MINGW32" (
  set "CONTITLE=MinGW x32"
  set "CONICON=mingw32.ico"
) else if "%MSYSTEM%" == "MINGW64" (
  set "CONTITLE=MinGW x64"
  set "CONICON=mingw64.ico"
) else if "%MSYSTEM%" == "UCRT64" (
  set "CONTITLE=MinGW UCRT x64"
  set "CONICON=ucrt64.ico"
) else if "%MSYSTEM%" == "CLANG64" (
  set "CONTITLE=MinGW Clang x64"
  set "CONICON=clang64.ico"
) else if "%MSYSTEM%" == "CLANG32" (
  set "CONTITLE=MinGW Clang x32"
  set "CONICON=clang32.ico"
) else if "%MSYSTEM%" == "CLANGARM64" (
  set "CONTITLE=MinGW Clang ARM64"
  set "CONICON=clangarm64.ico"
) else (
  set "CONTITLE=MSYS2 MSYS"
  set "CONICON=msys2.ico"
)
```

上面的代码是根据 `MSYSTEM` 变量显示不同的窗口标题，我在 CONTITLE 的内容加一个 loken，如下：

![msys2_shell-1-5](msys2_shell\msys2_shell-1-5.png)

------

```
if "x%MSYSCON%" == "xmintty.exe" goto startmintty
if "x%MSYSCON%" == "xconemu" goto startconemu
if "x%MSYSCON%" == "xdefterm" goto startsh

if NOT EXIST "%WD%mintty.exe" goto startsh
set MSYSCON=mintty.exe
:startmintty
if not defined MSYS2_NOSTART (
  start "%CONTITLE%" "%WD%mintty" -i "/%CONICON%" -t "%CONTITLE%" "/usr/bin/%LOGINSHELL%" --login !SHELL_ARGS!
) else (
  "%WD%mintty" -i "/%CONICON%" -t "%CONTITLE%" "/usr/bin/%LOGINSHELL%" --login !SHELL_ARGS!
)
exit /b %ERRORLEVEL%
```

上面的代码是最重要的，根据 MSYSCON 来启动终端，默认是用 mintty.exe ，所以只关注他就行，其他的终端不管。

如果没有定义 MSYS2_NOSTART 参数 就会执行以下命令：

![msys2_shell-1-6](msys2_shell\msys2_shell-1-6.png)

从上图可以看到，SHELL_ARGS 变量被传递给 mintty.exe了，也就是 -a -b -c 会传递 给 mintty.exe。

最后就调用 exit 退出了。

------

虽然最后控制权交给了 mintty.exe ，但是 MSYS2_PATH_TYPE 这些变量如何被 mintty.exe 使用的，还是不知道。

而且命令行参数 -mingw32 跟 -mingw64 在 CMD代码里面就只设置了一下 窗口标题跟图标，没有影响一些环境变量，非常奇怪。

我们可以使用 `mintty.exe --help` 来查看一下帮助文档。如下：

![msys2_shell-1-7](msys2_shell\msys2_shell-1-7.png)

------

我实验了一下，`msys2_shell.cmd` 这个脚本传递 -mingw32 跟 -mingw64 确实会影响 gcc ，如下：

![msys2_shell-1-8](msys2_shell\msys2_shell-1-8.png)

我创建了一个 **test.cmd** 脚本，内容如下：

```
set MSYSTEM=MINGW32
start "MinGW x32 loken" "C:\msys64\usr\bin\mintty" -i "/mingw32.ico" -t "MinGW x32 loken" "/usr/bin/bash" --login
```

就因为设置了环境变量 MSYSTEM ，所以进入了 mingw32 位环境。

所以我的判断是，msys2 的开发者，修改了 mintty 的代码，接受了 MSYSTEM 这些环境变量，因为CMD脚本里设置了 MSYSTEM 再调 mintty.exe，所以 mintty.exe 里面能拿到这些环境变量。然后做一些处理，例如设置默认的 gcc 是哪个gcc。

------

参考文章：

1，[批处理中setlocal enabledelayedexpansion的作用](https://www.cnblogs.com/ydhliphonedev/archive/2012/09/25/2702092.html)

2，[Windows脚本 - %~dp0的含义](https://www.cnblogs.com/smwikipedia/archive/2009/03/30/1424749.html)
