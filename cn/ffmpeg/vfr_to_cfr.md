# FFmpeg可变帧率转恒定帧率详解—ffmpeg.c源码分析

<div id="meta-description---">xxx</div>

在 `do_video_out()` 函数里面有一个比较复杂的逻辑，就是 `format_video_sync` 的策略，如下：

![1-1](vfr_to_cfr\1-1.png)

一共有四种 视频同步 策略：

1，VSYNC_VFR



2，VSYNC_CFR



3，VSYNC_VSCFR



4，VSYNC_PASSTHROUGH



5，VSYNC_DROP



---







默认情况 `video_sync_method` 是等于 `VSYNC_AUTO` 的，你也可以在命令行中使用 `-vsync` 对其进行修改。

上面的代码就是 `VSYNC_AUTO` 的策略。

1，



TODO：重复讲一下 VSYNC_VFR 这几个宏同步的作用。





```
format_video_sync = video_sync_method;
if (format_video_sync == VSYNC_AUTO) {
    if(!strcmp(of->ctx->oformat->name, "avi")) {
        format_video_sync = VSYNC_VFR;
    } else
        format_video_sync = (of->ctx->oformat->flags & AVFMT_VARIABLE_FPS) ? ((of->ctx->oformat->flags & AVFMT_NOTIMESTAMPS) ? VSYNC_PASSTHROUGH : VSYNC_VFR) : VSYNC_CFR;
    if ( ist
        && format_video_sync == VSYNC_CFR
        && input_files[ist->file_index]->ctx->nb_streams == 1
        && input_files[ist->file_index]->input_ts_offset == 0) {
        format_video_sync = VSYNC_VSCFR;
    }
    if (format_video_sync == VSYNC_CFR && copy_ts) {
        format_video_sync = VSYNC_VSCFR;
    }
}
ost->is_cfr = (format_video_sync == VSYNC_CFR || format_video_sync == VSYNC_VSCFR);

```



ffmpeg 可以通过

---

写作思路：

1，讲一下 VSYNC_AUTO 的算法规则。

```
format_video_sync = video_sync_method;
        if (format_video_sync == VSYNC_AUTO) {
```

