# sdl_audio_callback音频播放线程分析—ffplay.c源码分析

<div id="meta-description---">sdl_audio_callback音频播放线程分析</div>

音频播放线程是之前在 [audio_open()](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_open.html) 函数里面创建的，实际上就是**回调函数** （ `wanted_spec.callback`）。当用 SDL 打开音频硬件设备的时候，SDL 库就会创建一个线程，来**及时**执行回调函数 `sdl_audio_callback()`，至于 SDL 线程多久回调一次函数，这个我们不需要太关心，只要调 `SDL_OpenAudioDevice()` 函数的时候设置好相关参数即可。如下：

![1-1](sdl_audio_callback\1-1.png)

上图中，设置了每次回调取的样本数，设置了样本数就相当于设置了回调次数，`ffplay` 默认是 1秒钟最多回调 30 次 `sdl_audio_callback()` 函数。

------

`sdl_audio_callback()` 函数的参数如下：

```
static void sdl_audio_callback(void *opaque, Uint8 *stream, int len)
```

**1，**`void *opaque`，实际上就是之前设置的 `wanted_spec.userdata`，传递的是 `VideoState *is` 全局管理器。

**2，**` Uint8 *stream`，这个指针是 SDL 内部音频数据内存的指针，只要把数据拷贝到这个指针的地址，就能播放声音了。

**3，**`int len`，此次回调需要写多少字节的数据进去 `stream` 指针。

虽然 `len` 这个参数是 SDL 传递给我们的回调函数的，但是 `len` 其实是可以根据 `wanted_spec`的样本数，声道数，格式， 算出来的。

例如，`ffplay` 播放 [juren-5s.mp4](https://github.com/lokenetwork/FFmpeg-Principle/blob/main/ffplay/juren-5s.mp4) 的场景，命令如下：

```
ffplay -i juren-5s.mp4
```

在上面的命令中，`wanted_spec.samples` 会被赋值为 2048，也就是说每次回调，需要往 `stream` 指针写 2048 个采样。那2048个样本又是多少字节？

由于  `SDL_OpenAudioDevice()`  打开音频设备的时候写死了 16位格式 `AUDIO_S16SYS` ，所以一个采样是 2 个字节。然后因为 `juren-5s.mp4` 文件的音频是双声道的，每个声道都取2048个样本，那就是 2048 `*` 2 `*` 2 = 8192 字节。有兴趣可以打印一下 `sdl_audio_callback()` 里面的 `len` 变量，在本文命令下，一直都是 `8192 ` 字节。

------

`sdl_audio_callback()` 函数的流程图如下：

![1-2](sdl_audio_callback\1-2.jpg)



从上图的流程可以很容易看出， `sdl_audio_callback()` 函数干的事情，就是调 `audio_decode_frame()` 函数，把 `is->audio_buf` 指针指向要传输的音频数据。然后再调 `SDL_MixAudioFormat()` 把音频数据拷贝给 stream 指针指向的内存。这样 SDL 内部就会读 stream 指针指向的内存来播放。

虽然流程图画得比较简单，但是 `sdl_audio_callback()` 函数内部的代码逻辑是**非常复杂**的。下面我们一起来分析一下这里面的重点：

![1-3](sdl_audio_callback\1-3.png)

上图中的 `while(len>0){...}` 的逻辑简单概括就是从 `FrameQueue` 读取数据，不断写进入 SDL 的内存，直到写入了 `len` 大小的字节为止。

`audio_decode_frame()` 函数就是负责从  `FrameQueue` 读取 `AVFrame`，然后 把 `is->audio_buf` 指针指向 `AVFrame` 的 `data` ，如果经过重采样， `is->audio_buf` 指针会指向重采样后的数据，也就是 `is->audio_buf1`。

[audio_decode_frame()](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_decode_frame.html) 函数的**内部逻辑**是比较复杂的，本文不分析它的内部逻辑，只是给你讲一下它最后做到了什么事情，这也就是函数封装的意义。

`audio_decode_frame()` 函数最后做到的事情就是，把`is->audio_buf` 指针指向可以传输的音频数据，然后返回值代表可以传输的音频数据有多少字节。

至于这个音频数据是在哪个内存里面，我们外部调用不需要关心，这是它的内部实现。

如果  `audio_decode_frame()` 提取数据的时候发生错误。会返回 `-1`，然后把 `audio_buf` 指向 `NULL` 指针，这样会导致写入静音数据。

------

要理解上面的  `while(len>0){...}` 循环，有几个变量是必须要讲解一下的：

1，`is->audio_buf`，可以传输的音频数据的内存指针，指向内存的第一个字节。也就是开头。

2，`is->audio_buf_size`，可以传输的音频数据有多大，有多少个字节。

3，`is->audio_buf_index`，当前已经读取到第几个字节。

理解了这 3 个变量，就容易看懂上图中的 `while` 逻辑，就是不断拷贝数据到 SDL 的内存，

如果拷贝够了 `len` 字节，`is->audio_buf` 里面还有数据剩下来，就下一次回调来的时候继续拷贝剩下的数据。

如果当前 `is->audio_buf` 的数据不足以塞满 `len` 长度，就循环调 `audio_decode_frame()` 来提取数据，然后一直到 塞满 `len` 长度为止。

------

拷贝数据到 SDL 内存的时候有一个重点，**会根据音量来调不同的函数**，如下：

![1-4](sdl_audio_callback\1-4.png)

可以看到，如果直接 `memcpy` 拷贝数据，会是以最大音量进行拷贝。如果我们调整了音量，就会调 `SDL_MixAudioFormat()` 来拷贝数据。

因为 `SDL_MixAudioFormat()` 函数有调整音量的功能。

------

至此，`sdl_audio_callback()` 函数已经把 SDL 所需的音频数据全部拷贝给它了，就会跳出 `while` 循环，然后设置**音频时钟**。如下：

```
is->audio_write_buf_size = is->audio_buf_size - is->audio_buf_index;
    /* Let's assume the audio driver that is used by SDL has two periods. */
    if (!isnan(is->audio_clock)) {
        set_clock_at(&is->audclk,is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec,is->audio_clock_serial, audio_callback_time / 1000000.0);
        sync_clock_to_slave(&is->extclk, &is->audclk);
    }
```

上面的代码主要是设置音频的时钟，记录**当前**这一刻，音频流播放到哪里了。下面分析一下一些变量：

`is->audio_write_buf_size`  代表当前缓存里还剩多少数据没有拷贝给 SDL 。

`is->audio_clock` 这个变量是在 `audio_decode_frame()` 函数里面赋值的，因为 `audio_decode_frame()` 会从 `FrameQueue` 里面提取 一个`AVFrame`，而 `is->audio_clock`  记录的就是当这个 `AVFrame` 播放完之后，音频流所处的位置，也就是当 这个 `AVFrame` 播放完了，音频流 的 `pts` 是多少。。

理解了 `is->audio_clock` 就比较容易理解如何确定当前音频流播放到哪里了。

`is->audio_clock`  记录的是播放完那个 `AVFrame` 之后的 pts，但是此时此刻 只是把 这个 `AVFrame` 的内存数据拷贝给了 SDL，SDL 还没开始播放呢？

同时，SDL 还没播放的数据还有它内部的 `audio_hw_buf_size` 长度的数据，之前在《[audio_open函数分析](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_open.html)》说过：

> SDL线程并不是没有音频数据可以播放了才调 sdl_audio_callback() 来拿数据，而是他内部还剩 audio_hw_buf_size 长度的数据就会调 sdl_audio_callback() 来拿数据，是提前拿数据的。

所以，SDL 内部还剩 `audio_hw_buf_size` 字节，现在又来取了 `len` 字节，同时我们的 `audio_buf` 缓存还剩  `audio_write_buf_size`字节。总共有 3 块内存等待播放，而这 3 块内存播放完之后的 `pts` 就是 `is->audio_clock` ，如下：

![1-6](sdl_audio_callback\1-6.jpg)

从上图可以看到，我们已经知道播放完这 3 块内存之后，音频流的位置，那就可以**反向**求音频流当前的播放时刻。而 `len` 实际上就是等于 `audio_hw_buf_size`，只是换了下名字，所以可以直接用 `audio_hw_buf_size` 乘以 2，所以就产生了下面的这句代码：

```
音频流当前的播放时刻 = is->audio_clock - (double)(2 * is->audio_hw_buf_size + is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec
```

这里要注意 `is->audio_clock` 的单位是秒。至此，就可以正确设置音频流当前播放到哪里了。

代码里有一句注释，也讲解了为什么要乘以 2。

```
/* Let's assume the audio driver that is used by SDL has two periods. */
```

它假设了 SDL 打开的设备里面是有两段缓存的，也就是  `audio_hw_buf_size` + `len`。

------

最后的那句代码主要是用音频时钟来同步一下外部时钟，如下：。

```
sync_clock_to_slave(&is->extclk, &is->audclk);
```

时钟是用来记录当前的播放时刻的，方便做音视频同步，同步相关的分析，请看以下文章：

1，《[音视频同步基础知识](https://ffmpeg.xianwaizhiyin.net/ffplay/basic_sync.html)》

2，《[FFplay视频同步分析](https://ffmpeg.xianwaizhiyin.net/ffplay/video_sync.html)》

3，《[FFplay音频同步分析](https://ffmpeg.xianwaizhiyin.net/ffplay/audio_sync.html)》

4，《[FFplay外部时钟分析](https://ffmpeg.xianwaizhiyin.net/ffplay/external_sync.html)》

