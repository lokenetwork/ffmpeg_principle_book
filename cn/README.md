# FFmpeg原理介绍
<div id="no_ads"></div>

<div id="meta-description---">本书《FFmpeg原理》主要讲解 FFmpeg 原理性的知识，前面几章是音视频开发，流媒体技术的基础，后面的章节主要讲解如何搭建 FFmpeg 各种调试环境，同时提供 FFmpeg API 函数的使用示例，最重要的是 分析 ffmpeg.c 跟 ffplay.c 的源码逻辑。</div>

<img src="img/cover.jpg" alt="通过 postermywall 制作" style="zoom: 33%;" />

本书《FFmpeg原理》主要讲解 FFmpeg 原理性的知识，前面几章主要讲解一些音视频开发的基础知识，例如原始数据 YUV 跟 RGB，封装格式 FLV 跟 MP4 ，压缩编码的基本概念，还有封装格式分析。

前面几章是音视频开发，流媒体技术的基础，后面的章节主要讲解如何搭建 FFmpeg 各种调试环境，同时提供 FFmpeg API 函数的使用示例，最重要的是 分析 ffmpeg.c 跟 ffplay.c 的源码逻辑。

虽然官方没有提供太详细的 API 函数文档教程，但是官方提供了 `ffmpeg.c` 文件 ，大部分的 API 函数使用方法，都在这个文件的源码里面。

基本上你用到的 **FFmpeg 命令行 **的所有功能，都是在 `ffmpeg.c` 里面实现的。包括 转换封装格式，转换编码格式，合并视频流，各种滤镜功能，都能在 `ffmpeg.c` 里面找到对应的 API 函数的用法。

本书会用大量章节来分析  `ffmpeg.c` 里面的内部逻辑，让读者能从 整体上 理解 FFmpeg API 的使用。对 FFmpeg API 形成系统的理解之后，即便新版本的 FFmpeg 修改了 API 函数的用法，你也能从 `ffmpeg.c` 里面快速学会新版本的API函数的用法。

**提示1：本书的所有图片都是高清图，请直接用新标签打开高清图即可。**

**提示2：本书第一版是以 FFmpeg-n4.4.1 源码来分析写作**

------

如果想订阅本书的内容更新，可以关注 公众号 **FFmpeg弦外之音** 或者订阅 TG 的 [Channel](https://t.me/ffmpeg_principle)  

<div align=center>
    <img src="./img/gongzhonghao.jpg" style="" />
</div>

由于笔者的水平有限， 加之编写的同时还要参与开发工作，文中难免会出现一些错误或者不准确的地方，恳请读者批评指正。

------

《FFmpeg原理》是一本半公益的音视频入门书籍，有 70% 的内容可以免费阅读，如果你觉得这些内容对您有帮助，可以通过微信打赏我。

<div align=center>
    <img src="img/pay.jpg" style="width:33%"/>
</div>

捐赠列表如下：

- [镕铭微电子NETINT](https://www.netint.cn/)
- 王哥（王*I）
- 王化春
- 肖志宏
- 夏楚
- 尹同民
- 明月惊鹊
- 余生爱静
- 阿羽
- 松松呀
- 投降吧对手
- 灰调光影
- chan
- 风宝
- 洋洋和甜甜
- 孙东东
- 符至渊·纸鸢
- 胡必腾
- sharpbai
- 王彬
- 末
- 雷祥
- 黎海
- 黄子文
- Zero
- Hansen King
- 黄金蛇
- 陈湘
- 徐春明
- 邵航
- 山林不向四季起誓
- 鲁日樟
- wencoo
- 上善若水
- 陈昱翔
- 山追言成
- 无名大侠 W

------

同时感谢以下热心网友对本书做出的贡献，包括内容审校，提供代码思路，等等。

松松呀，第十人称，若水，明月惊鹊，天之魂，江梦梁@动游科技，quink，Mo Cuishle，AyaseEri，徒步青云，Rainy，阿小步，Uker，Neko，蔡和伦，周礼，XN，梧桐樹下，Thomson，白衣，老徐，黄金蛇，贾献华，Hansen King，aaykbcn，尹同民，D1ngkai，黎海，人中赤兔马中吕布，银河也是河，司雨寒，野良丶猫，执念

如有遗漏贡献者，请联系我添加

------

版权声明：本书内容不允许任何形式的转载。但可以简单引用部分段落，必须标明出处。

