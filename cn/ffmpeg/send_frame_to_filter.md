# send_frame_to_filters滤镜处理—ffmpeg.c源码分析

<div id="meta-description---">send_frame_to_filters()函数主要的职责是调 av_buffersrc_add_frame_flags() 把 AVFrame 往 滤镜里发送，但是如果滤镜没打开就会用 configure_filtergraph 打开滤镜。</div>

`send_frame_to_filters()` 函数会把 `AVFrame` 发送给输入流关联的所有入口滤镜进行处理。但是它的代码是比较简单的，如下：

![1-0](send_frame_to_filter\1-0.png)

可以看到 `send_frame_to_filters()` 只不过就是在循环调 `ifilter_send_frame()` 函数而已。

`send_frame_to_filters()` 函数只有一个重点，就是在调 `ifilter_send_frame()` 之前，创建多了一份 `AVFrame` 的引用，为什么需要这样做呢？

答：因为发送完 `AVFrame` 给滤镜之后，就会调 `av_frame_unref()` 把引用减一，滤镜容器是异步线程，所以为了让滤镜容器里面能拿到 `AVFrame` 的内存，需要先把引用加一，要不发送完之后就立即释放内存，滤镜线程就拿不到内存了，如下：

![1-1](send_frame_to_filter\1-1.png)

---

下面来分析一下 `ifilter_send_frame()` 函数。

 `ifilter_send_frame()` 函数首先需要对比 `InputFilter ifilter` 跟 `AVFrame` 的格式，采样率，宽高等参数，如果不一样，就把 `need_init` 设置为 1，如下：

![1-2](send_frame_to_filter\1-2.png)

由于 `ifilter->format` 一开始是设置为 -1，所以第一次 `need_init` 肯定会设置为 1。

`need_reinit` 为 1，就会用 `AVFrame` 对 `ifilter` 进行赋值，我把这个 `ifilter_parameters_from_frame` 调用成为 `InputFilter` 初始化。

```
if (need_reinit) {
    ret = ifilter_parameters_from_frame(ifilter, frame);
    if (ret < 0)
    	return ret;
}
```

注意一个滤镜容器（FilterGraph）是有多个 `InputFilter`，这只是对其中一个进行初始化。

当 `FilterGraph` 里面还有 `InputFilter` 没有初始化的时候，就不能把 `AVFrame` 发送给滤镜，只能先写进去队列（`ifilter->frame_queue`），如下：

![1-3](send_frame_to_filter\1-3.png)

`InputFilter` 的 `format` 不等于 -1 就代表初始化完成了。

当 `FilterGraph` 里面所有的 `InputFilter` 都初始化完成了，就会调 `configure_filtergraph()` 来配置滤镜链，里面会打开滤镜容器。如下：

![1-4](send_frame_to_filter\1-4.png)

 `configure_filtergraph()` 执行完之后，滤镜容器就打开了，就可以调 `av_buffersrc_add_frame_flags()` 往滤镜发送 `AVFrame` 了。如下：

![1-5](send_frame_to_filter\1-5.png)

细心的读者可能会发现了，在未打开滤镜容器之前，写进去队列（`ifilter->frame_queue`）的数据没处理么？

其实是有处理，就在  `configure_filtergraph()` 里面，它里面打开滤镜之后，就会把队列缓存的数据都往滤镜里面发送。

具体介绍，推荐阅读《[configure_filtergraph配置滤镜容器](https://ffmpeg.xianwaizhiyin.net/ffmpeg/configure_filtergraph.html)》

---

做下小总结， `ifilter_send_frame()` 函数主要的职责是调 `av_buffersrc_add_frame_flags()` 把 `AVFrame` 往 滤镜里发送，如下：

```
ret = av_buffersrc_add_frame_flags(ifilter->filter, frame, AV_BUFFERSRC_FLAG_PUSH);
```

但是，代码不一定就会跑到 `av_buffersrc_add_frame_flags()` 。前提是滤镜容器已经打开了。

如果没有打开，就会先初始化 `InputFilter`，等所有 `InputFilter` 都初始化之后，再打开滤镜容器。

