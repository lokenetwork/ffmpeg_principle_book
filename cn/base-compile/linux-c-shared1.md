# Linux环境显式使用动态库—编译链接基础知识

<div id="meta-description---">Linux环境显式使用动态库</div>

前一篇文章中  libstar.so 是动态链接到 libc 库的。如果 zeus.c 需要静态链接 libc ，会导致什么问题呢？

假设这个 libstar.so 是第三方提供的动态库，我们没有这个 libstar.so 动态库的源码。但是我们自身的项目 zeus 需要使用静态libc，就会这样进行链接，如下：

```
gcc -o zeus zeus.o -L/usr/local/star/lib -Wl,-Bstatic -lc -Wl,-Bdynamic -lstar
```

上面的命令没有使用 -static 选项，因为它会导致所有的库都以静态方式链接，所以使用了原始的 Wl 选项。

上面的命令虽然能正常生成 zeus ，但是运行会直接报错，如下：

![linux-c-shared-1-5](linux-c-shared\linux-c-shared-1-5.png)

这时候，如果如果我们有  libstar.so 的源代码，是不是可以设置 生成  libstar.so 的时候 静态链接到 libc。尝试一下，命令如下：

```
gcc -fPIC -shared -o libstar.so sun.o moon.o earth.o -static 
# 或者
gcc -fPIC -shared -o libstar.so sun.o moon.o earth.o -Wl,-Bstatic -lc 
```

上面两条命令其实都会报错，如下：

![linux-c-shared-1-6](linux-c-shared\linux-c-shared-1-6.png)

所以怎么办呢？这种情况，目前有两种解决方法，第三方库 libstar 不要编译出动态库，直接用静态库就完事了。生成静态库不会进行链接。

但是，如果你没有 libstar 的源代码，只有第三方公司提供 libstar.so 动态库。而你的程序为了兼容，又需要静态链接 libc.a。

这时候可以使用 **显式调用动态库**的方法，把 zeus.c 的代码改成下面的：

```
#include <stdio.h>
#include <dlfcn.h>
int main()
{
    int (*sun_rotate)();
    void* star_so = dlopen("/usr/local/star/lib/libstar.so",RTLD_LAZY);
    if (!star_so){
        printf("Open Error:%s.\n",dlerror());
        return 0;
    }

    sun_rotate = dlsym(star_so,"sun_rotate");

    printf("zeus do something \r\n");

    (*sun_rotate)();

    return 0;
}
```

执行编译命令：

```
gcc -o zeus zeus.o -ldl -static
```

![linux-c-shared-1-7](linux-c-shared\linux-c-shared-1-7.png)

虽然会报一个 warning ，但是可以正常跑。这样，我们的 zeus 程序就是静态链接 libc，而  libstar.so 是动态链接libc。

因为这是两个C运行时库不一样，一个是动态的 ，一个是静态的，用的时候要小心，可能你调 libstar.so 拿到的变量。在 libc 静态库函数里面不能用。

------

参考资料：

1，[《linux下动态链接库的显式调用和隐式调用》](https://blog.csdn.net/wojiushiwoba/article/details/53035883)
