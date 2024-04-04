# 如何设置编码器参数—FFmpeg API教程

<div id="meta-description---">如何设置编码器参数，编码器 （encode）的参数也是分为 通用部分 跟 私有部分。通用部分是指大部分编码器都有的属性，通用部分的参数是在 avcodec_options 变量里面的</div>

编码器 （encode）的参数也是分为 **通用部分** 跟 **私有部分**。通用部分是指大部分编码器都有的属性，例如**码率**就是通用的。通用部分的参数是在 `avcodec_options` 变量里面的，如下：

![1-1](encode_args\1-1.png)

也可以通过 `ffmpeg.exe -h > t.txt` 来查看通用部分，如下：

![1-2](encode_args\1-2.png)

编码器私有部分的参数可以通过以下命令查询：

```
ffmpeg.exe -h decoder=libx264
ffmpeg.exe -h encoder=libx264
```

![1-3](encode_args\1-3.png)

------

无论是通用还是私有属性，都是使用 `AVDictionary` 来设置的，就是最后一个参数 `AVDictionary **options`，如下：

```
int avcodec_open2(AVCodecContext *avctx, const AVCodec *codec, AVDictionary **options);
```

下面通过一个例子来演示如何设置编码器参数，本文代码可在 [GitHub](https://github.com/lokenetwork/FFmpeg-Principle/tree/main/encode_args) 下载：

![1-4](encode_args\1-4.png)

上图中，演示了**两种**设置编码器属性的方法，一种是直接赋值各个字段，例如 `pix_fmt`， `color_range` 就是采用第一种方法。

第二种是通过 `AVDictionary` 来设置，传递进去 `avcodec_open2()` 函数里面即可，**libx264** 的 `preset` 默认是 `medium` 的，我把他改成 `superfast` （超级快）。

> [!NOTE]
> `avcodec_open2()` 函数如果使用了 `AVDictionary` 的值，就会删除掉，例如 `avcodec_open2()` 函数执行完之后，`preset` 这个值就会从 `AVDictionary` 里删除。

最后使用完之后，需要调 `av_dict_free()` 来释放 `AVDictionary` 的内存。

------

解码器参数的设置 跟编码器参数设置是类似，照葫芦画瓢就行。解码器的参数通常比较少。
