# audio_open函数分析—ffplay.c源码分析

<div id="meta-description---">audio_open函数分析</div>

`audio_open()` 的作用，就如同它的名字那样，就是**打开音频设备**。流程图如下：

![1-1](audio_open\1-1.jpg)



------

SDL 库播放音频数据有两种方式。

1，调用层定时往 SDL 接口塞数据。

2，设置SDL回调函数，让 SDL 来主动执行回调函数来取数据。

第二种方式的实时性更好，`ffplay` 也是用的第二种。

------

先介绍一下  `audio_open()`  函数的各个参数，如下：

```
static int audio_open(void *opaque, int64_t wanted_channel_layout, int wanted_nb_channels, int wanted_sample_rate, struct AudioParams *audio_hw_params)
```

1，**void *opaque**，传递给 SDL 回调函数的参数。

2，**wanted_channel_layout**，**wanted_nb_channels**，**wanted_sample_rate**，希望用 这样的采样率，声道数，声道布局打开音频硬件设备。

3，**struct AudioParams *audio_hw_params**，实际打开的音频硬件设备的音频格式信息。这里的 hw 是 Hardware 的意思，也就是硬件，不过不是指硬件编解码加速，而是指打开的硬件设备。

这些参数是这样的，**想要的音频格式** 跟 **实际打开的音频格式** 不一定一样，例如，MP4 里面的音频流是 48000 采样的，肯定想要用 48000 的格式打开音响设备，这样才能听到最好的音质。但是难免有些音响设备太差，只支持 44100。

------

下面来分析一下 `audio_open()` 函数的重点代码，如下：

![1-2](audio_open\1-2.png)

第一个重点就是这两个数组变量，`next_nb_channels[]` 跟 `next_sample_rates[]`，这两个变量会让人摸不着头脑，即便看了后面的 `while` 逻辑，也不太容易看懂他们的含义。

现在我直接剧透这量个变量的作用。首先是 `next_nb_channels[] `，这 其实是一个map表，声道切换映射表。举个例子，如果音响设备不支持 7 声道的数据播放，肯定不能直接报错，还要尝试一下**其他声道**能不能成功打开设备吧。这个**其他声道**就是  `next_nb_channels[] `。

- next_nb_channels[7] = 6，从7声道切换到6声道打开音频设备
- next_nb_channels[6] = 4，从6声道切换到4声道打开音频设备
- next_nb_channels[5] = 6，从5声道切换到6声道打开音频设备
- next_nb_channels[4] = 2，从4声道切换到2声道打开音频设备
- next_nb_channels[3] = 6，从3声道切换到6声道打开音频设备
- next_nb_channels[2] = 1，从双声道切换到单声道打开音频设备
- next_nb_channels[1] = 0，单声道都打不开音频设备，无法再切换，需要降低采样率播放。
- next_nb_channels[0] = 0，0声道都打不开音频设备，无法再切换，需要降低采样率播放。

而 `next_sample_rates[]` 变量存储的仅仅是采样率，当切换**所有声道**都无法成功打开音频设备，就需要从 `next_sample_rates[]` 取一个比**当前**更小的采样率来尝试。

所以可以看到 `next_sample_rates[]` 的元素都是从小到大的。

------

`audio_open()` 函数的**第二个重点**是 声道数，声道布局等参数的校验，如下：

![1-3](audio_open\1-3.png)

`ffplay` 经常会校验  **声道布局 跟 声道数是否一致**，可能是担心用户在命令行输入错误的参数。

此外还需注意一下 `next_sample_rate_idx` 变量，这个变量存的是比 **想要打开的音频采样率** 更小一点的采样率的索引。

```
while (next_sample_rate_idx && next_sample_rates[next_sample_rate_idx] >= wanted_spec.freq)
        next_sample_rate_idx--;
```

------

`audio_open()` 函数的**第三个重点**是 设置打开参数，设置回调函数 ，如下：

![1-4](audio_open\1-4.png)

可以看到，`ffpaly` 是**写死了采样格式**，写死成了 `AUDIO_S16SYS` 格式。也就是说，无论MP4里面的音频采样格式是 64 还是 32 位，还是其他，统统都会先提前转成 `AUDIO_S16SYS`，然后再丢给 SDL 播放。

另外，`wanted_spec.samples` 的作用是告诉 SDL 每次回调取多少样本数来播放，实际上就是控制 SDL 每秒调用回调函数的次数。

因为播放采样率是固定的，例如 48000。也就是音频设备每秒要播放 48000 个样本，如果调一次 `sdl_audio_callback` 只能取4800个样本，那他一秒内就需要调 10 次 `sdl_audio_callback`。

同理，只要固定了调用次数，也就能计算出 SDL 每次回调**需要**取多少样本数。而 `ffplay` 就固定了每秒回调 `SDL_AUDIO_MAX_CALLBACKS_PER_SEC` （30次）。

 `wanted_spec.samples` 的计算方式有点难懂，如下：

```
wanted_spec.samples = FFMAX(SDL_AUDIO_MIN_BUFFER_SIZE, 2 << av_log2(wanted_spec.freq / SDL_AUDIO_MAX_CALLBACKS_PER_SEC));
```

首先，`SDL_AUDIO_MIN_BUFFER_SIZE` 宏是 512，所以最小值是 512。但是 后面的用 `av_log2` 取指数，又 `<<` 位移，是干什么呢？

**答：**`wanted_spec.freq` 是采样率，而 `SDL_AUDIO_MAX_CALLBACKS_PER_SEC` 代表 SDL 每秒调多少次回调函数。两者相除就能得出每次回调需要读取多少个**样本数**。这个逻辑非常容易理解。

`av_log2()` 函数是求对数，也就是 2 乘以 自身**多少次**等于括号里面的结果，这个多少次会进行取整操作的，`av_log2()` 不会返回小数。

然后 `2 <<` 位移只是想把 **样本数数量** 变成 **2 的指数**。这是 SDL 文档建议的，推荐阅读《[SDL_OpenAudioDevice](https://wiki.libsdl.org/SDL_OpenAudioDevice)》。

**疑问：**打开音频设备失败之后，会进行降低采样率播放操作，`wanted_spec.freq` 可能会变，但是 `wanted_spec.samples` 没有跟着变。我也不清楚会不会有问题。

------

`audio_open()` 函数的**第四个重点**是 用 `while` 循环不断**尝试**各种 声道 跟 采样的组合来打开音频硬件设备，如下：

![1-5](audio_open\1-5.png)

比较难懂的就是 `next_nb_channels[FFMIN(7, wanted_spec.channels)]`，大家可以假设一下 `wanted_spec.channels` 如果是 7， `next_nb_channels[FFMIN(7, wanted_spec.channels)]` 等于多少呢？，推算过程如下：

```
next_nb_channels[FFMIN(7, 7)]
```

```
next_nb_channels[7] = 6
```

可以看到，如果 7 声道打开失败，就会切换成 6 声道进行尝试，跟上面说的那个映射是一致的。

而 `next_sample_rates[next_sample_rate_idx--]` 就是进行降低采样率，再尝试打开音频设备，之前说过，`next_sample_rate_idx` 是比 想要的采样率更小一点的值。

注意他的那句日志 `No more combinations to try, audio open failed`，其实这句日志就说明了这段代码的逻辑，就是**尝试各种组合**。

------

提醒：`SDL_OpenAudioDevice()` 函数打开音频设备之后，回调函数是不会立即执行的，需要调 `SDL_PauseAudioDevice()` 来启动音频设备，这样回调函数才会执行。

```
if ((ret = decoder_start(&is->auddec, audio_thread, "audio_decoder", is)) < 0)
	goto out;
SDL_PauseAudioDevice(audio_dev, 0);
break;
```

------

最后一个重点就是，打开硬件设备成功之后，要**记录一下**是以什么样的采样率，声道数等等格式打开硬件设备的。当 `AVFrame` 与 硬件音频格式不一致时候需要进行转换。

![1-6](audio_open\1-6.png)

`audio_hw_params` 是指硬件设备参数的意思，我个人看到 `hw` 第一反应会以为是硬件编解码加速，其实不是。

硬件参数有两个字段需要特别讲解一下 ：

1，`audio_hw_params->frame_size`，一个音频样本占多少内存。

2，`audio_hw_params->bytes_per_sec`，`audio_hw_params->freq` 数量的音频样本占多少内存。也就是一秒钟要播放多少内存的音频数据。

这两个字段都是用`av_samples_get_buffer_size()` 来计算的。

**注意 ：`audio_hw_params` 实际上就是 `is->audio_tgt` ，只是在函数内部换了个名字而已。**

------

至此，`audio_open()` 函数就讲解完毕了，但是还要注意一下这个函数的返回值，如下：

![1-7](audio_open\1-7.png)

**audio_open()** 的返回值，最后是赋值给 `is->audio_hw_buf_size` 的，如下：

![1-8](audio_open\1-8.png)

我们可以看一些，SDL 头文件对这个 `spec.size` 的定义，如下：

```
typedef struct SDL_AudioSpec
{
    int freq;                   /**< DSP frequency -- samples per second */
    SDL_AudioFormat format;     /**< Audio data format */
    Uint8 channels;             /**< Number of channels: 1 mono, 2 stereo */
    Uint8 silence;              /**< Audio buffer silence value (calculated) */
    Uint16 samples;             /**< Audio buffer size in sample FRAMES (total samples divided by channel count) */
    Uint16 padding;             /**< Necessary for some compile environments */
    Uint32 size;                /**< Audio buffer size in bytes (calculated) */
    SDL_AudioCallback callback; /**< Callback that feeds the audio device (NULL to use SDL_QueueAudio()). */
    void *userdata;             /**< Userdata passed to callback (ignored for NULL callbacks). */
} SDL_AudioSpec;
```

`Audio buffer size in bytes` 指的是 SDL 内部音频数据缓存的大小，代表 SDL线程执行 `sdl_audio_callback()` 的时候，SDL 硬件内部还有多少字节音频的数据没有播放。

没错，SDL线程并不是没有音频数据可以播放了才调 `sdl_audio_callback()` 来拿数据，而是他内部还剩 `audio_hw_buf_size` 长度的数据就会调 `sdl_audio_callback()` 来拿数据，是提前拿数据的。这个 概念 以及 `audio_hw_buf_size` 变量 都特别重要，后面会用到。
