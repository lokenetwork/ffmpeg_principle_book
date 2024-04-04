# FFmpeg是如何调整输出帧率的—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

待写





注意讲解一下这段日志代码。

```
if (debug_ts) {
        av_log(NULL, AV_LOG_INFO, "filter -> pts:%s pts_time:%s exact:%f time_base:%d/%d\n",
               frame ? av_ts2str(frame->pts) : "NULL",
               frame ? av_ts2timestr(frame->pts, &enc->time_base) : "NULL",
               float_pts,
               enc ? enc->time_base.num : -1,
               enc ? enc->time_base.den : -1);
    }
```

