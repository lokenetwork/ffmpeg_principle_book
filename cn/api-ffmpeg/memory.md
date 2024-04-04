# FFmpeg内存管理—FFmpeg API教程

<div id="meta-description---">FFmpeg 是一个 C程序的项目，C语言是需要手动管理内存的。内存管理有一个技巧，只要你分清楚这个变量是在栈上，还是在堆上的就可以了。在栈上的变量不需要手动释放，而在堆上的变量需要自己释放。</div>

**FFmpeg** 是一个 C程序的项目，C语言是需要手动管理内存的。内存管理有一个技巧，只要你分清楚这个变量是在栈上，还是在堆上的就可以了。在栈上的变量不需要手动释放，而在堆上的变量需要自己释放。

下面通过一个实例， 通过 反汇编 来讲解一下 **栈变量 为什么不需要自己释放**，代码如下：

```
int add_something(int a, int b, int c) {
    int d = 4;
    int e = 5;
    int f = 6;
    return a + b + c + d + e + f;
}

int sub_something(int a, int b, int c) {
    int g = 7;
    int h = 8;
    int j = 9;
    return a + b + c - g - h - j;
}

int main() {
    int a = 1;
    int b = 2;
    int c = 3;
    int total;
    total = add_something(a, b, c);
    total = sub_something(a, b, c);
    return total;
}
```

首先，**一个线程在大多数系统上会默认有1M的栈内存**，这个大小可以通过配置修改。也就是一个线程创建的时候，默认就会先申请1M的栈内存。

有些同学可能会觉得，1M 是不是有点小，如果定义了太多的局部变量，用完这 1M 怎么办？用完了就会报错，如果你的程序访问了 1M 地址之外的内存，就会报 segment fault 错误。

补充：在 Windows 系统下，可能会根据需要动态调整线程的栈内存大小，就是不够用了会动态扩大，具体我需要测试一下验证。

我们知道，函数调用会先 压栈，然后再弹栈，但是还有一个地方也会用到栈内存，就是局部变量，你每创建一个局部变量，栈就会变得更小一点。

首先，栈内存的访问通常是由两个寄存器控制的。如下：

1，`esp` ，栈寄存器，存储的是当前的栈地址，会随着代码运行不断地变化

2，`ebp`，栈顶寄存器，存储的是栈顶的地址，变化较少。

现在反汇编一下上面的代码，如下：

![1-1](memory\1-1.png)

可以看到，进入 `add_something` 函数的时候，入口那里，有个 `sub` 指令，**这个指令会直接切走一块栈内存**，给 局部变量用。也不仅仅是给局部变量用，也有其他用途。

我们注意看，现在 EBP 寄存器的值是 0x00B5F630，也就是 d 变量的地址是 0x00B5F630 - 4 = 0x00B5F62C ，为什么我们不需要用 free 函数释放 0x00B5F62C 地址的内存呢？我们继续断点，进入 `sub_something` 函数，就清楚了，如下：

![1-2](memory\1-2.png)

可以看到，g 变量的内存地址也是 0x00B5F630 - 4 = 0x00B5F62C，所以当 `sub_something` 执行的时候，之前 `add_something` 函数里面的 局部变量  d，e，f 指向的内存就会被覆盖。

所以栈内存，是一个可以不断重复使用的内存。所以局部变量不需要释放，只要离开了函数的作用域，之前的局部变量内存就会被重新覆盖使用。

扩展知识：函数调用，压栈，弹栈，会不断修改恢复 `ebp` 跟 `esp`。**所以栈内存能不断被重复使用**。`main` 也是一个函数，`main` 的外面其实还有一层函数调它。

------

下面开始介绍一下 FFmpeg 跟内存相关的函数，如下：

1，`av_frame_alloc`，创建一个 `AVFrame` 结构的堆内存。对应 `av_frame_free`

2，`av_packet_alloc`，创建一个 `AVPacket` 结构的堆内存。对应 `av_packet_free`

3，`avformat_alloc_context`，创建一个 `AVFormatContext` 结构的堆内存。对应 `avformat_close_input`  或者  `avformat_free_context`

4，`avcodec_alloc_context3`，创建一个 `AVCodecContext` 结构的堆内存。对应 `avcodec_close` 或者 `avcodec_free_context`

5，`av_malloc`，申请内存。对应  `av_free` 跟 `av_freep` ，其中 `av_freep` 函数接受的是二级指针，所以会把指针置为 NULL。这样做可以减少错误。

