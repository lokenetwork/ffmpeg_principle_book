# video_refresh视频播放线程分析—ffplay.c源码分析

<div id="meta-description---">video_refresh视频播放线程分析</div>

视频播放线程就是 `main` 主线程，对于 `FFplay` 播放器，就是在 **主线程** 里面播放视频流的，如下：

![1-1](video_refresh\1-1.png)

如上图所示，`event_loop()` 会不断用  `refresh_loop_wait_event()` 函数检测是否有键盘事件发生，如果有键盘事件发生，  `refresh_loop_wait_event()` 就会返回，然后跑到 `switch{event.type}{...}` 来处理键盘事件。

如果没有键盘事件发生， `refresh_loop_wait_event()` 就不会返回，只会不断循环，不断去播放视频流的画面。如下：

![1-2](video_refresh\1-2.png)

`refresh_loop_wait_event()` 函数里面的重点是 `remaining_time` 变量，这个变量是什么意思呢？

`remaining_time` 变量的默认值是 0.01（REFRESH_RATE），在 `video_refresh()` 里面可能会改变 `remaining_time` 的值，如下：

```
if (time < is->frame_timer + delay) {
	*remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
	goto display;
}
```

上面的代码中计算出来的 `remaining_time` 代表要播放下一帧视频，还需要等待多少秒。可以看到用了 `FFMIN` 取最小值。

所以`remaining_time` 变量的含义是，要播放下一帧视频，还需要等待多少秒。或者说要过多久才去检查一下是否可以播放下一帧。

上图中有个 `av_usleep(remaining_time)` ，就是为了避免 `while` 循环，过于频繁去检查下一帧是否可以播放。

举个例子：

如果视频流的帧率是24帧每秒，也就是每隔0.04秒播放一帧数据，那  `video_refresh()` 里面大部分情况会把 `remaining_time` 赋值为 0.01，每隔 0.01s 去检查下一帧是否可以播放了。

如果视频流的帧率是200帧每秒，也就是每隔0.005秒播放一帧数据，那  `video_refresh()` 里面大部分情况会把 `remaining_time` 赋值为 0.005，0.005 实际上就是播放下一帧视频，还需要等待多长时间。



---

真正播放视频流的函数是 `video_refresh()` 函数，流程图如下，绿色是默认不会执行的逻辑，不用关注。

![1-3](video_refresh\1-3.jpg)

从代码的角度看， `video_refresh()` 总共只有 4 段逻辑，如下图：

![1-3](video_refresh\1-3.png)

1，当主时钟是外部时钟的逻辑。（第一个绿色框）

2，当需要播放音频的波形图的时候。（第二个绿色框）

3，渲染视频流的数据到窗口上。注意变量 `is->force_refresh` ，这个变量是控制是否渲染SDL画面的，渲染完之后会恢复成 0 。（第三个红色框）

4，打印音视频的同步信息到控制台上。（第四个红色框）

绿色的是不重要的，默认情况是不会跑进去绿色的逻辑。最重要的是红色的圈圈。

---

`video_refresh()` 函数里面最重要的就是 `if (is->video_st)){...}` ，因为就是在这块代码里面控制视频流的播放。

下面来**宏观**看一下 `if (is->video_st)){...}` 这段代码里面的逻辑，如下：  

![1-5](video_refresh\1-5.png)

首先里面有两个 `label`，分别是 `retry` 跟 `display`。可以看到，如果 `FrameQueue` 队列没有数据，就会立即跑到 `display` 的位置调 `video_display()` 函数显示画面。 

`video_display()` 这个函数主要是负责把 视频帧 `AVFrame` 的数据渲染到 `SDL_Texture`（纹理）上面。

非常值得注意的是 `video_display()` 取的是上一帧视频来播放的，里面调用的函数是 `frame_queue_peek_last()` 。

这里说的上一帧视频，我指的是当前窗口画面正在显示的视频帧，只要它显示在窗口上了，它就是**上一帧**了，而**下一帧**代表还没播放显示的帧。

这里读者可能会疑惑，如果 `video_display()` 取的是上一帧，那怎么行？画面就一直是上一帧，画面就不会动。

答：没错，所以我缩进起来的逻辑 `else{...}`，会调 `frame_queue_next()` 来偏移读索引，这样就会导致**下一帧**变成了**上一帧**。

`video_display()` 里面也有显示音频波形图的逻辑，但不是本文重点。

------

所以从**宏观上**，`video_refresh()` 函数有两个逻辑。

**第一**，`FrameQueue` 队列无数据可读，取上一帧来渲染SDL窗口，通常是因为**调整了窗口大小**才会执行 `video_display()` 重新渲染。如下：

![1-6](video_refresh\1-6.png)

注意 `is->force_refresh` 这个变量只有是 1 才会 执行  `video_display()` 。执行完 `video_refresh()` 之后 `is->force_refresh` 会重新变成 0。

**第二**，`FrameQueue` 队列有数据可读，就会跑进去 `else{...}` 的逻辑，`peek` 一个帧，看看是否可以播放，如果可以播放，设置  `is->force_refresh` 为 1，然后再 执行  `video_display()`  渲染画面。

------

下面再来具体分析一下缩进起来的  `else{...}` 的逻辑，如下：

![1-7](video_refresh\1-7.png)

可以看到，上面一个循环，不断从 `FrameQueue` 队列读取数据，直至读到跟 `is->videoq.serial` 序列号一致的 `Frame`，这样做可以把失效的 `Frame` 通通丢弃。因为快进快退的时候，会导致队列里面缓存的 `Frame` 失效。

------

`else{...}` 里面接下来的重点是如下：

![1-8](video_refresh\1-8.png)

`is->frame_timer` 这个变量出现的频率非常高，所以需要先讲解一下 `frame_timer` 这个变量的含义。 

`frame_timer` 可以理解为 窗口正在显示的帧 的播放时刻，就是说这帧是何时开始播放的，从何时开始显示到窗口上的。

其实还有另一个变量也是用来记录 视频帧的播放时刻的，那就是视频时钟 `clock`。 `is->frame_timer`  跟 `clock` 是同时更新的，如下：

![1-9](video_refresh\1-9.png)

只不过 `frame_timer` 的时间单位是系统时间，而 `clock` 的时间单位是 `pts`。两者有不同的用途。

`av_gettime_relative()` 函数可以简单理解为获取系统时间，只不过它是从一个任意位置开始的系统时间。



---

现在已经了解了 `is->frame_timer` 变量的含义，再来分析一下 `frame_timer` 的第一次赋值，以及后续变化的过程逻辑。

**1，** `is->frame_timer` 变量第一次赋值的地方是在 下面的代码，如下：

```
time= av_gettime_relative()/1000000.0;
...
if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
      is->frame_timer = time;
```

这几句代码的本意是，如果 当前系统时间 比 当前帧的开始播放时刻 大  0.1 （AV_SYNC_THRESHOLD_MAX），就会重置 `frame_timer` 为当前系统时间。

这种情况可能是因为某些原因，视频流播放线程卡顿了很久，导致 当前时间与 `frame_timer` 差距过大，也就是上一帧显示得太久了。

但是这几句代码，也是**第一次赋值** `frame_timer` 的代码，`frame_timer` 一开始是 0 ，所以 `time` 减去 `is->frame_timer` 必然大于 `AV_SYNC_THRESHOLD_MAX`

。所以，`frame_timer` 就会在这里第一次被赋值为系统时间。

------

**2，**如果不进行快进快退，`frame_timer` 就会一直累加 `delay`，如下：

![2-1](video_refresh\2-1.png)

`vp_duration()` 函数是用来获取 窗口正在显示的帧 需要显示多长时间的。

在 `video_refresh()` 里面，比较容易混淆上一帧，当前帧，下一帧的概念。我在本文说的上一帧就是当前帧的意思，`lastvp` 就是窗口正在显示的帧，但是 `last` 直译过来，就是上一个的意思。对于这些概念，我尽量讲得具体一些。

`compute_target_delay()` 函数里面会进行**视频同步**操作，会把 `last_duration` 减少或者增加，然后返回 `delay`。所以 `delay` 可能会比 `last_duration` 大一点或者小一点，也可能 `delay` 等于 `last_duration` 。

如果视频流比音频流播放慢了，那 `delay` 会比 `last_duration` 小一些，如果视频流比音频流播放快了，那 `delay` 会比 `last_duration` 大一些。

`last_duration` 代表当前帧本来，**本来**需要显示多长时间。当前帧是 指 窗口正在显示的帧。

`delay` 代表 当前帧实际，**实际**应该显示多长时间。

举个例子，1/24 帧的视频流，每帧固定显示 0.04s，当音频跟视频播放完全同步的时候，`last_duration` 跟 `delay` 都会是 0.04。

但是当视频比音频快了 0.05s 的时候，那 `delay` 就会从 0.04 变成 0.08，翻倍了，拉长当前视频帧的播放时间来等待音频流追上来。

这个翻倍是 `compute_target_delay()` 函数的算法规则，本文不打算讲解视频同步的更多算法细节，只是简单讲一下 `last_duration` 跟 `delay` 变量的关系。

**3，**如果进行了快进快退，`is->frame_timer` 就会重新赋值为系统时间。

```
if (lastvp->serial != vp->serial)
   is->frame_timer = av_gettime_relative() / 1000000.0;
```

------

通常情况下，event_loop 函数是每隔 0.01s 来检测是否可以播放下一帧视频，检测代码如下：

```
time= av_gettime_relative()/1000000.0;
if (time < is->frame_timer + delay) {
	*remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
	goto display;
}
```

上面说过 `frame_timer` 为 当前帧 的播放时刻，`delay` 代表当前帧实际应该显示多长时间，如果当前帧还没显示完，就会直接 `goto` 到 `display` 的位置，这时候 `force_refresh` 是 0 ，所以不会调 `video_display()` 进行渲染，什么都不会做，直接退出 `video_refresh()` 了。

------

还有一个重点是暂停状态下的逻辑，如下：

![2-2](video_refresh\2-2.png)

为什么暂停状态下 要直接跳到 `display` ？暂停状态下直接 `return`，或者在 入口检查一下，直接退出 `video_refresh()` 就行了。暂停状态下，肯定不需要播放视频，渲染窗口的啊？

解答一下：暂时状态下，是不需要再播放视频流的下一帧的，但是有可能需要重新渲染窗口。因为，因为暂停状态下，你可以调整 `ffplay` 的窗口大小的，之前说过，当前窗口大小变了，就需要去上一帧来重新渲染SDL。

这就是 暂停状态下  跳到 `display` 的意义，调整了窗口大小，就会导致 `force_refresh` 置为 1，跳到 `display` 的时候，就会跑进去 `if` 条件，执行 `video_display()` 函数。

------

`video_refresh()` 里面有两段检查视频帧时间的逻辑。

1，检查当前帧是否已经显示完毕。（前面的代码逻辑）

2，检查要播放的下一帧是否已经过了播放时间。（如下图）

![2-3](video_refresh\2-4.png)

上面这种情况是这样的，假设视频流的每帧应该只显示 0.04s，但是由于系统卡顿，第三帧显示了 0.1s秒，这个值大于了 is->frame_timer + duration，所以就会导致丢帧，会把地第 4 帧丢弃，转而去播放第 5 帧。

`is->frame_drops_late` 是统计从 `FrameQueue` 读取数据的时候的丢帧数量。

------

当两段时间检测逻辑都通过了之后，就可以显示要播放的帧的，如下：

```
frame_queue_next(&is->pictq);
is->force_refresh = 1;
```

这两句代码非常精妙，`frame_queue_next()` 会偏移读索引，**所以导致了下一帧变成了当前帧**，或者说导致下一帧变成了上一帧。本文我说的当前帧就是上一帧。而 `force_refresh` 置为 1 后，就可以调 `video_diaplay() `了。

------

下面再分析最后一个重点，就是 `video_diaplay()` 的判断逻辑，如下：

```
display:
        /* display picture */
        if (!display_disable && is->force_refresh && is->show_mode == SHOW_MODE_VIDEO && is->pictq.rindex_shown)
            video_display(is);
```

`display_disable` 默认是 0，可以通过命令行参数改变，主要就是控制窗口不要显示画面，无论是视频帧，还是音频波形都不显示。

`is->force_refresh`  变量有两种情况会置为 1，**一是**当下一帧可以播放的时候，**二是**当窗口大小产生变化的时候。

`is->show_mode` 默认就是 `SHOW_MODE_VIDEO` 。

最后一个条件 `is->pictq.rindex_shown` 是重点，有点不太容易看出来他为什么在 `if` 加上这个判断。

之前在《[FrameQueue队列分析](https://ffmpeg.xianwaizhiyin.net/ffplay/frame_queue.html)》讲过，`rindex_shown` 的初始值是 0。只有在插入第一帧到 `FrameQueue` 的时候， `rindex_shown` 变量才会变成 1。

所以，加上  `is->pictq.rindex_shown` 条件，就是为了防止 `FrameQueue` 一帧数据都没有，就调了 `video_display()`。

从代码逻辑上看，当 `FrameQueue` 队列为空的时候，是会直接跳到 `diaplay` 的，所以如果不在 `if` 加上  `is->pictq.rindex_shown`，会有问题。

![2-6](video_refresh\2-6.png)

总结：`if` 条件里面的`is->pictq.rindex_shown` ，是用来防止播放线程启动运行得太快，`FrameQueue` 什么都没有的时候，就调 `video_display()`;

------

注意，最后渲染完画面之后，`is->force_refresh` 会重新赋值 为 0。

![2-5](video_refresh\2-5.png)

------

后面的 `if (show_status){...}` 是输出控制台的日志，显示音频时钟跟视频时钟的时间差，如下图：

![1-4](video_refresh\1-4.png)

------

`video_refresh()` 视频播放线程目前就讲解完毕了，里面有一个变量忽略了，就是 `is->step` ，这个变量是 **FFplay** 的逐帧播放功能，默认是 0 。推荐阅读《[FFplay逐帧播放分析](https://ffmpeg.xianwaizhiyin.net/ffplay/step.html)》
