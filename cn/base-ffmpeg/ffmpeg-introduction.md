# FFmpeg介绍—FFmpeg基础

<div id="meta-description---">FFmpeg 是一个可以处理音视频的软件，功能非常强大，主要包括，编解码转换，封装格式转换，滤镜特效。同时也支持 各种网络协议，支持 RTMP ，RTSP，HLS 等高层协议的推拉流，也支持更底层一点的TCP/UDP 协议推拉流。</div>

FFmpeg 是一个可以处理音视频的软件，功能非常强大，主要包括，编解码转换，封装格式转换，滤镜特效。同时也支持 各种网络协议，支持 RTMP ，RTSP，HLS 等高层协议的推拉流，也支持更底层一点的TCP/UDP 协议推拉流。

在多平台系统方面，FFmpeg 的兼容性也优势显著，FFmpeg 可以在 windows，Linux，Mac，ios，android 等等操作系统上运行。

因此，可以说 FFmpeg 是音视频领域的瑞士军刀。在多个公司都有使用，例如 Google 的 chrome 里面就使用了 FFmpeg 的库。还有 Youtube，Facebook，以及国内的各种做音视频产品的公司，只要他做音视频，95% 都会用到 FFmpeg。

但是截止 2022 年，FFmpeg 还是只有 命令行，没有GUI 图像界面，所以对使用者有一定的门槛。

------

FFmpeg 是一个开源项目，起始于2000年，截止 2022 年，已经走过 22 年，在这过程中，FFmpeg 社区经历过一次分裂。2011年的时候，一群 FFmpeg 开发者由于对项目管理者（不是Fabrice Bellard）不满，，而另立山头，创建了 Libav 项目。

我个人觉得，他们可能是对 **当时** 的FFmpeg 代码混乱不满。Libav 的代码架构更清晰一些。不过那是很多年前的事情了。目前 FFmpeg 的代码架构还是很不错的。

Libav 项目经历了几年的发展，还是没有发展下去，最后 Libav 的成果代码，被合并到 FFmpeg 里面，git 提交记录保留了 Libav 开发者的名字。

推荐阅读 [《FFmpeg 跟 Libav 的渊源》](https://trac.ffmpeg.org/wiki/Using%20libav*)

------

FFmpeg 的 用户主要有 3 类。

**1，社区开发者**，直接写 FFmpeg 代码的，截止2022年，一共有100多个 Maintainer（主要开发者）。

**2，FFmpeg API 库使用者**，FFmpeg 提供了很多的动态库给上层开发者调用。这类开发者主要是调 API，偶尔会提交一下代码反馈给 社区。

**3，FFmpeg 命令行使用者**，这类用户通常不会写 C/C++，但是具备一点的电脑操作知识，主要在电影，电视台这些行业。这类用户只会使用 FFmpeg 的命令行，比较厉害的会写 shell脚本 跟 batch批处理 来 使用 FFmpeg。

