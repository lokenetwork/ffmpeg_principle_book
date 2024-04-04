# 编译运行多个Java文件

Java 里面的多文件编译 相对于 C++ 来说简单很多，Java 项目的构建其实不需要你手动执行 链接 这个过程，只需要把 `.java` 后缀的文件全部编译成 `.class` 后缀的文件即可，下面用一个例子来讲解一下 Java 的多文件编译。

本文的代码可以在 [Github](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/java_demo) 上进行下载，下载之后把 java_demo 文件夹放到 D 盘。

这个项目有两个文件，`Main.java`，`MyMatch.java`。`MyMatch.java` 类会提供一个 `sum()` 函数来累计数组里面的值。

编译步骤如下：

1，先编译 `MyMatch.java`，命令如下：

```
javac.exe MyLib\MyMath.java
```

执行完之后，会生成 `MyMath.class` 二进制的字节码文件，如下：

![1-1](java_many\1-1.png)

---

2，编译 `Main.java`，命令如下：

```
javac.exe Main.java
```

生成的文件如下：

![1-2](java_many\1-2.png)

---

然后就可以直接运行 `Main.class` 了，如下：

```
java.exe Main
```

![1-3](java_many\1-3.png)

---

可以看到，整个编译，然后运行的过程比较简洁，虽然 `Main.java` 依赖于 `MyMath.java` ，但是编译的时候，你不需要手动指定 `MyMath.java`，他的编译器 `javac.exe` 会根据代码里面的语法自动找到 `MyMath.java`。

如果我们把  `MyMath.java` 跟  `MyMath.class` 都删除，那编译 `Main.java` 的时候就会报错，如下：

![1-4](java_many\1-4.png)

如果编译出来 `Main.class` 之后，再把 `MyMath.class` 删除，那运行 `Main.class` 也会报错，如下：

![1-5](java_many\1-5.png)

因此无论是编译，还是运行的时候，都是依赖 `MyMath.class` 的。

---

至此，我们已经了解了在 Java 里面多个文件的编译，以及运行。Java 比 C++ 的构建过程更加简单，他内部就是根据 `import` 与 `package` 关键词来定位到依赖的库，或者说依赖的类。

Java 大部分框架都是由 多个 java 文件组成了，Java 的生态工具链非常封装，有很多库，以及不同场景的框架可以使用，下面就让我们来简单使用一下 spring 框架。请看下一篇文章。
