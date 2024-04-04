# 编译运行单个Java文件

众所周知，Java 语言的特点是跨平台，拥有垃圾回收，自动管理内存等特点，Java 的语义比 C++ 更容易理解，可以让开发者更专注于业务层开发。

Java 代码的后缀是 `.java` ，这些代码首先通过 `javac` 编译器翻译成 java 字节码。新建一个文件 `Hello.java`，内容如下：

```
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello FFmpeg Principle");
    }
}
```

然后使用以下命令编译这个文件：

```
javac.exe Hello.java
```

![1-1](java_hello\1-1.png)

上图中生成的 `Hello.class` 文件就是 java 的字节码，java 字节码也可以说是 jvm 指令集，这是一种类似于汇编指令的东西。

`Hello.class` 是一个二进制文件，需要借助 `javap` 命令来查看 `Hello.class` 里面的内容，如下：

**`javap` 命令里面的 p 代表 print。**

```
javap.exe -c Hello.class
```

![1-2](java_hello\1-2.png)

上图中的 jvm 指令是可以直接被 jvm 虚拟机直接**解析**运行的，jvm 虚拟机其实 就是 `java` 命令，如下：

```
java.exe Hello
```

![1-3](java_hello\1-3.png)

`java.exe` 里面是原生的 CPU 指令，因为我是 AMD 的 CPU，所以 `java.exe` 里面是 x86 的机器指令。

---

虽然 jvm 指令一开始是解析运行的，但是从 1996 年，Sun 公司为 java 平台提供了 Just-In-Time（JIT）编译器，可以把 jvm 指令翻译成 本地的 CPU 指令。

JVM 虚拟机会选择性地把 经常运行的 jvm 指令翻译成 本地指令，然后下次再执行这段代码，就不是解析执行了，而是直接运行本地指令，性能会有所提升。

整个过程如下：

![1-4](java_hello\1-4.png)

上图出自美团的《[基本功 | Java即时编译器原理解析及实践](https://tech.meituan.com/2020/10/22/java-jit-practice-in-meituan.html)》

---

因此，jvm 指令集是一种虚拟的指令集，一种中间代码。有比较多的基础设施都会采用这种**中间代码**的设计模式，来让软件达到更好的移植性，例如 NVIDIA 的 PTX 指令集也是一种中间代码，PTX 最终会翻译成 SASS 指令来运行，NVIDIA 也用了 JIT 技术。

由于 jvm 指令集与汇编指令非常类似，所以可以用**极低的计算成本**把 jvm 指令翻译成 本地的 x86 汇编指令 或者 arm 汇编指令 等等 原生的 CPU 指令集。

一门语言的语法越复杂，或者说越高级的语言，把它翻译成 原生 CPU 指令集的计算成本就越大。











