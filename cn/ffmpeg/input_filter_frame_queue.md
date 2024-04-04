# InputFilter的frame_queue队列介绍—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

待写。

---

参考：https://www.xianwaizhiyin.net/?p=723

3，strcut InputFilter 里面的 frame_queue 是一个 AVFrame 的临时存储区，为什么要临时存储，是因为 FilterGraph 里面的所有 InputFilter 都初始化完成才能 往 某个filter 里面写 AVframe ，ffmpeg 是这样判断 InputFilter是否初始化完成的，InputFilter::format 不等于 -1 就是 初始化完成了。具体实现在 ifilter_has_all_input_formats() 函数里。如果 A InputFilter 初始化完成了，B InputFilter 没初始化完成，就不会往 A 的 InputFilter::filter 写数据，而是先写到 A 的 InputFilter::frame_queue，后面再从 InputFilter::frame_queue 里拿出来，写到 InputFilter::filter。部分代码如下：

