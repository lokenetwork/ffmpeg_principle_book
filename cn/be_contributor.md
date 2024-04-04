# 如何成为FFmpeg开发者
<div id="meta-description---">但是如果想继续提升 FFmpeg 的开发能力，是必须有一些项目实战的，空看源码是没有意义的，必须找到应用场景。看这段源码是为了解决什么问题。对于刚熟悉 FFmpeg 的朋友来说，项目实战有两种方式。</div>

当你看完本书的时候，如果能熟悉使用各种 `FFmpeg` 的 API 函数，了解播放器原理，能 debug 深入探索 `ffmpeg.c` 的逻辑。

**那恭喜您，您已经算是一个 FFmpeg 的开发者了。**

但是如果想继续提升 FFmpeg 的开发能力，是必须有一些项目实战的，空看源码是没有意义的，必须找到应用场景。看这段源码是为了解决什么问题。

对于刚熟悉 FFmpeg 的朋友来说，项目实战有两种方式。

**1，**做音视频开发兼职，有些音视频QQ群，微信群，有时候会有一些老板或者工程师想付费咨询一些 FFmpeg 的问题。这时候如果你觉得自己能解决他们的问题，就可以尝试联络一下。

不过对于有正式工作的朋友，不建议接工作量太大的活，可以接那种100~300左右的小问题咨询，没帮别人解决问题不收钱就行，也没有多少心理负担。

这种小兼职，通常可以晚上下班，或者周末时间进行，既提升了自己的音视频开发能力，又能赚到一些钱。

**2，**参与 FFmpeg 社区开发，FFmpeg 社区每天都会有各种人提各种需求，各种bug。如果您有时间，可以尝试去解决一下。

不过参与  FFmpeg 社区开发更多是一种长期的投资，本身是没有收入的，需要一些坚持。

----

本文主要分享一下如何参与 FFmpeg 社区开发。

FFmpeg 是一个历史悠久的开源项目，起始于2000年，那时候还没有 **GitHub**，**Sourceforge** 等等代码托管平台。

所以 FFmpeg 的开发者自己搭建了 Git 服务器，虽然现在 Github 上也有 FFmpeg ，但是在 GitHub 是看不到 issue 等等讨论信息的，FFmpeg 在 GitHub 只有一个代码镜像。

FFmpeg 官方的 Git 址如下：

```
https://git.ffmpeg.org/ffmpeg.git
```

我们可以把这个仓库拉下来，看一下有多少个分支，多少个 tag。

```
git clone https://git.ffmpeg.org/ffmpeg.git
git branch -a
git tag
```

---

FFmpeg 在 GitHub 没有 issue，即便你提了 issue，开发者也不会去看，那他们去哪里查看一些问题反馈呢？

答：FFmpeg 开发者是用 [trac](https://trac.edgewall.org/) 开源项目做了一个网站来管理 bug，网站地址 ：trac.ffmpeg.org

如果你想提 bug， 需要先登陆这个网站进行注册，如下：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-0-1.png)

注册完成之后，新建任务单即可提 bug，如下图：

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-2-1.png)

推荐阅读官方的 《[bugreports教程](https://ffmpeg.org/bugreports.html)》

---

FFmpeg 社区对于 Git 的使用是有一些规范的，推荐阅读《[git-howto](https://ffmpeg.org/git-howto.html)》，如果不熟悉 Git 操作可以先看一遍 [《廖雪峰 Git 》](https://www.liaoxuefeng.com/wiki/896043488029600)

---

FFmpeg 社区开发功能，修复bug，讨论新版本，都是非常公开透明的，你可以通过 email 了解到所有开发者在干什么。

FFmpeg 社区是通过邮件来沟通的，有一个 [Mailing Lists](https://ffmpeg.org/contact.html#MailingLists)，如下：

![1-1](be_contributor\1-1.jpg)

可以看到，一共有 5 个组，分别用来讨论不同的场景业务的。

**1，**ffmpeg-devel ：这个 channel 是讨论 FFmpeg 的源码开发的，新功能，新特性，新的编解码器的开发，都会在这个 channel 讨论。订阅这个 channel 就能看到别人在讨论什么。没错，FFmpeg 是任何人都能参与开发，讨论的。

**2，**ffmpeg-user：ffmpeg.exe，ffplay.exe 这些命令行工具的使用讨论。

**3，**libav-user ： 讨论 ffmpeg dll 库的 api 函数的使用。

**4，**ffmpeg-cvslog : FFmpeg 源码的更新修改通知，订阅这个channel，发布了新特性功能就会通知你。

**5，**ffmpeg-trac : 提 bug 的channel。

如果你使用 FFmpeg 遇到一些 bug，可以在 [ffmpeg-trac](https://ffmpeg.org/pipermail/ffmpeg-trac) 上搜索，可能别人也遇到了。这样会比用搜索引擎更快，因为有些 bug 还没被搜索引擎收录。

---

不经常用邮件沟通的朋友可能不太了解什么是 `MailingLists`。`MailingLists` 其实跟你直接发邮件给别人，或者别人直接发邮件给你没什么太大区别。

Email 电子邮箱有两个接受角色，**收件人** 与 **抄送人**。

- 收件人：直接接收邮件的人，代表这封邮件面向的读者。可以是零个到多个。
- 抄送人：可以理解为微博的 at 功能，即告知。可以是零个到多个。

MailingLists 的用处是，你可以订阅某个邮件地址，也可以理解为订阅某个channel，例如 [ffmpeg-trac@avcodec.org](mailto:ffmpeg-trac@avcodec.org)，别人发往 [ffmpeg-trac@avcodec.org](mailto:ffmpeg-trac@avcodec.org) 地址的邮件，就会往你的邮箱**抄送**一份，你发往 [ffmpeg-trac@avcodec.org](mailto:ffmpeg-trac@avcodec.org) 的邮件，也会往别人的订阅邮箱**抄送**一份。

推荐阅读 《[mailing-list-faq](https://ffmpeg.org/mailing-list-faq.html)》

---

下面介绍一下如何定阅 `MailingLists`。打开 [ffmpeg-trac 订阅](https://lists.ffmpeg.org/mailman/listinfo/ffmpeg-trac) ，在 上面填上自己的邮箱即可，如下图：

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3.png" alt="img" style="zoom:50%;" />

新加入的订阅，想要看到之前的问答讨论怎么办？可以 在 [Archives](https://ffmpeg.org/mailing-list-faq.html#toc-Archives) 里面找到之前的问答讨论。

也可以通过 google 搜索引擎指定网站搜索，如下：

```
site:lists.ffmpeg.org/pipermail/ffmpeg-devel/ "search term"
```

---

FFmpeg 还有一个[实时聊天室](https://kiwiirc.com/nextclient/irc.libera.chat/?channels=#ffmpeg,#ffmpeg-devel)，可以在上面提问题。比较实时能得到解答。

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-4-1.png" alt="img" style="zoom:50%;" />

上图中的 `ffmpeg` channel 是讨论命令行使用的，`ffmpeg-devel` channel 是讨论开发问题的。

---

**问：**如何应用一个patch 补丁到自己本地？

**答：**`git am`，推荐阅读 《[FetchingPatchworkPatches]( https://trac.ffmpeg.org/wiki/FetchingPatchworkPatches)》

---

**问：**如何修复别人给 FFmpeg 提的bug？

**答：**在 trac [找到对应的](https://trac.ffmpeg.org/) bug 记录，修复这个bug后，把代码打包成一个 patch。然后通过邮件发送 patch ，patch 发送成功了的话，会出现在 [patchwork.fmpeg.org](https://patchwork.fmpeg.org) 网站上的。 

---

**问：**如何生成一个 patch，如何用邮件发送 patch ？

**答：**下面介绍一下我是修复一个非常小的小角落，我在 trac 上提了一个小bug，代号是 [9634](https://trac.ffmpeg.org/ticket/9634) 

先执行以下命令，拉取 FFmpeg 源代码代码，切换到 4.4 版本分支。

```
git clone https://git.ffmpeg.org/ffmpeg.git
cd ffmpeg
git checkout remotes/origin/release/4.4
```

我加了以下代码，加了一个判断 `#if CONFIG_AVRESAMPLE`，这个宏判断他们之前忘记加了。

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-2.png)

然后执行以下命令提交到本地仓库

```
git add fftools/cmdutils.c 
git commit -m 'fftools/cmdutils.c: add if CONFIG_AVRESAMPLE (ffmpeg version 4.4)'
```

再执行下面的命令把上一次的提交打一个patch，可以看到生成了个 patch文件。

```
git format-patch HEAD^
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-5.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-6.png)

然后用邮件发送这个 patch 文件到  [ffmpeg-devel@ffmpeg.org](mailto:ffmpeg-devel@ffmpeg.org)  。Git 如何发送邮件推荐阅读[《如何使用git send-email》](https://www.jianshu.com/p/fb09b6a87533)

我的 Git 邮箱配置如下：

```
[sendemail]
        smtpserver = smtp.qq.com
        smtpuser = loken2022@qq.com
        smtppass = 你的QQ邮箱授权码
        from = 2338195090@qq.com
        to = ffmpeg-devel@ffmpeg.org
        confirm = always
```

配置好发送邮件环境，执行以下命令：

```
git send-email -1
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-7.png)

发送邮件成功之后，在 [patchwork.ffmpeg.org](https://patchwork.ffmpeg.org/) 就能看到提交记录了，如下：

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-8.png" alt="img" style="zoom:50%;" />

由于 FFmpeg 提交 patch 只能在最新的 commit 提交，今天是 2022年 2月，最新的 commit 是5.0 的，我提交的 patch 是基于 4.4 的，5.0之后已经没有这个bug了。所以 我提的这个 patch 基本不会有人理我。

因为版本大改之后，某些 API 会改变，或者舍弃。对于旧版本的bug，应该如何提交反馈。

旧版本的bug，可以在 [trac.ffmpeg.org](https://trac.ffmpeg.org/) 上面提交一个 bug 反馈，然后把相关的 patch 修复文件，上传到附件里面。

如下，这是是一个修复 [cuda的patch](https://trac.ffmpeg.org/ticket/9019)。

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-9.png" alt="img" style="zoom:50%;" />
