# process_input处理输入文件—ffmpeg.c源码分析

<div id="meta-description---"> process_input() 主要是从输入文件读取一个 AVPacket，然后丢给 process_input_packet() 函数处理这个 AVPacket</div>

在刚开始的时候，滤镜容器都是未打开的状态，所以会从所有的输入流里面选出一个还未解码出来 `AVFrame` 的输入流，来进行处理，如下：

![1-1](process_input\1-1.jpg)

`process_input()` 函数的定义如下：

```
/*
 * Return
 * - 0 -- one packet was read and processed
 * - AVERROR(EAGAIN) -- no packets were available for selected file,
 *   this function should be called again
 * - AVERROR_EOF -- this function should not be called again
 */
static int process_input(int file_index){
	...省略代码...
}
```

可以看到，注释写得很清楚，一共有 3 个返回值。

1，返回 0 ，代表已经读取了一个 `AVPacket`，并且处理了。

2，返回 `AVERROR(EAGAIN)`，代表**暂时**没有 `AVPacket` 可以读取出来进行处理。实际上就是 `av_read_frame()` 返回 `AVERROR(EAGAIN)`，`process_input()` 就会返回 `AVERROR(EAGAIN)`

3，返回 `AVERROR_EOF`，代表所有的输入文件，都处理完成，不能再次调 `process_input()`

---

`process_input()` 函数的重点代码如下：

**第一个重点：处理文件循环以及文件结束，**如下：

![1-2](process_input\1-2.png)

由于本文是初步讲解 ffmpeg.exe 转换器的转码过程，所以暂时不讲结束的处理逻辑，读者可以后面再看《FFmpeg结束处理》一文。

---

**第二个重点：统计数据，**如下：

![1-3](process_input\1-3.png)

注意上面的 `ist->discard`，这个字段不是 API 数据结构的，而是 `ffmpeg.c` 自己定义的，**属于用户层定义的丢弃规则**。如果你设置的是 `AVStream` 的 `discard`，那根本不可能读到这个流的 `AVPacket`。

---

第三个是非重点：处理时间戳回环问题，如下：

```
if(!ist->wrap_correction_done && is->start_time != AV_NOPTS_VALUE && ist->st->pts_wrap_bits < 64){
        int64_t stime, stime2;
      	...省略代码..
}
```

在一些流格式，例如 TS，会有时间戳环回的问题，`int64` 的最大值是 xxx，超过这个大小会环回，`ffmpeg.exe` 需要处理环回的逻辑。

---

第四个是非重点：处理 side data，如下：

```
/* add the stream-global side data to the first packet */
if (ist->nb_packets == 1) {
    for (i = 0; i < ist->st->nb_side_data; i++) {
    	...省略代码..
    }
}
```

一些非重点的代码，不用特意去看，用到去看就行了。

---

第五个是非重点：处理时间戳，如下：

    if (pkt->dts != AV_NOPTS_VALUE)
        pkt->dts += av_rescale_q(ifile->ts_offset, AV_TIME_BASE_Q, ist->st->time_base);
    if (pkt->pts != AV_NOPTS_VALUE)
        pkt->pts += av_rescale_q(ifile->ts_offset, AV_TIME_BASE_Q, ist->st->time_base);
    
    if (pkt->pts != AV_NOPTS_VALUE)
        pkt->pts *= ist->ts_scale;
    if (pkt->dts != AV_NOPTS_VALUE)
        pkt->dts *= ist->ts_scale;

上面的 ts_offset 默认是 0，ts_scale 默认是 1，所以如果你的命令行参数没用到对应的选项，这些代码跟没运行一样。

---

**第六个是重点：对时间戳进行纠正**，如下：

![1-4](process_input\1-4.png)

上面的 `ifile->last_ts` 记录的是上一次读取出来的 `AVPacket` 的 `dts` 时间。

---

**第七个是重点：pts 附加上文件的当前时间**，如下：

![1-5](process_input\1-5.png)

注意，上面的 `ifile->duration`，不是指输入文件的时长，而是这样的，怎么说呢，这个 `duration` 默认是 0， 如果文件循环了一次，`duration` 就是输入文件的时长，如果循环了两次，`duration` 就是输入文件的时长乘以 2，这是为了 `-loop` 选项服务的代码。

---

**第八个是重点：还是对时间戳就行修正**，如下：

![1-6](process_input\1-6.png)

上面的 `ist->next_dts` 是 由上一个 `pkt` 的 `dts` + `duration` 计算出来的，`duration` 是根据帧率算的。

可以看到，第六 ，第八 个重点都是修正时间戳，但是计算 `delta` 的方式是不一样的。

**还有一个小彩蛋，就是 `if (debug_ts) {..}`，在纠正时间之前会打印一次，纠正之后，也会打印一次，如果你想看看纠正时间的日志，就可以打开 debug_ts 这个调试选项**

---

最后就是调 `process_input_packet()` 函数来处理读取到的 `AVPacket`。

```
process_input_packet(ist, pkt, 0);
```

因此，`process_input()` 函数的总体流程如下：

![2-1](process_input\2-1.jpg)

提示：**我一般会用圆角矩形来表示这个逻辑是有条件的**，也就是这个逻辑可能会执行，也可能不会执行。。







