# FFmpeg强制关键帧分析

<div id="meta-description---">xxx</div>

待写

---

写作思路：

1，好像，强制编码器生成 I 帧，直接设置个属性就可以了。

```
if (forced_keyframe) {
            in_picture->pict_type = AV_PICTURE_TYPE_I;
            av_log(NULL, AV_LOG_DEBUG, "Forced keyframe at time %f\n", pts_time);
        }
```

