# FFplay键盘功能介绍—ffplay.c源码分析

<div id="meta-description---">FFplay 播放器在运行过程中，可以通过一些键盘的按键触发一些功能</div>

FFplay 播放器在运行过程中，可以通过一些键盘的按键触发一些功能，有哪些按键可用，可以通过以下命令查看。

```
ffplay --help
```

![1-1](event_loop\1-1.png)

**1，**`q` 键，`ESC` 键。退出 `ffplay` 播放器，`ffplay` 播放器播放完mp4之后，默认会停在最后一帧画面，不会退出的。推荐阅读《[FFplay退出分析](https://ffmpeg.xianwaizhiyin.net/ffplay/exit.html)》

**2，**`f` 键，切换全屏。

**3，**`p` 键，空格键，暂停播放。推荐阅读《[FFplay暂停分析](https://ffmpeg.xianwaizhiyin.net/ffplay/pause.html)》

**4，**`m` 键，切换静音。

**5，**`9` 键，`0` 键，`/` 键，`0` 键，调整音量。

**6，**`w` 键，切换视频滤镜，`ffplay` 播放器的命令行是可以指定多个视频滤镜，然后按 `w` 键切换查看效果的。

**7，**`s` 键，开启逐帧播放模式。可以一帧一帧播放视频。推荐阅读《[FFplay逐帧播放详解](https://ffmpeg.xianwaizhiyin.net/ffplay/step.html)》

**8，**左右键，快进快退 10 秒。

**9，**上下键，快进快退 60 秒。推荐阅读《[FFplay跳转时间点播放](https://ffmpeg.xianwaizhiyin.net/ffplay/jump.html)》

**10**，右键，根据右键的位置，进行跳转时间点播放。

------

这些按键的处理代码，全部是在 `event_loop()` 函数里面的，如下：

![1-2](event_loop\1-2.png)
