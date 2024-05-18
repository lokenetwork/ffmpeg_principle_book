# How to Become an FFmpeg Developer

<div id="meta-description---">
However, to further enhance the development skills of FFmpeg, engaging in practical projects is imperative. Mere observation of the source code without real-world application renders it meaningless; it's essential to identify practical use cases and understand the purpose behind examining the source code. For those who are new to FFmpeg, there are two primary approaches to participating in practical projects.
</div>

Once you've completed this book, if you're adept at using a variety of _FFmpeg_ API functions, grasp the fundamentals of media player principles, and can debug and explore the logic of `ffmpeg.c` in depth.

However, to further improve your FFmpeg development skills, you need some practical projects. Simply reading the source code is meaningless; you must find application scenarios and understand the problems you're trying to solve with that particular code.

For those who are just getting familiar with FFmpeg, there are two ways to gain practical experience:

**1. Freelancing in audio and video development:** There are some audio and video QQ groups and WeChat groups where employers or engineers occasionally seek paid consultations for FFmpeg-related issues. If you feel confident in solving their problems, you can try contacting them.

However, for those with full-time jobs, it's not recommended to take on large-scale projects. You can accept small consultations for around $15-$50. If you can't solve their problem, you don't charge, so there's no significant burden.

These small freelance jobs can often be done after work or on weekends, allowing you to improve your audio and video development skills while earning some extra income.

**2. Participating in FFmpeg community development:** The FFmpeg community receives various requests and bug reports every day. If you have the time, you can try to address some of them.

However, participating in FFmpeg community development is more of a long-term investment, as it doesn't provide direct income and requires some persistence.

---

This article mainly shares how to participate in FFmpeg community development.

FFmpeg is a long-standing open-source project that started in 2000, before the existence of code hosting platforms like GitHub and SourceForge.

Therefore, FFmpeg developers set up their own Git server. Although there is an FFmpeg repository on GitHub now, you won't find issues or discussion information there; it only serves as a mirror of the codebase.

The official Git address for FFmpeg is:

```
https://git.ffmpeg.org/ffmpeg.git
```

We can clone this repository and check the number of branches and tags:

```
git clone https://git.ffmpeg.org/ffmpeg.git
git branch -a
git tag
```

---

Since FFmpeg doesn't use issues on GitHub, and developers won't see them even if you create them, where do they look for feedback?

**Answer:** FFmpeg developers use the open-source [trac](https://trac.edgewall.org/) project to manage bug reports and discussions. The website address is: trac.ffmpeg.org

If you want to report a bug, you need to register on this website first, as shown below:

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-0-1.png)

After registration, you can create a new ticket to report the bug, as shown below:

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-2-1.png)

It's recommended to read the official [bugreports tutorial](https://ffmpeg.org/bugreports.html).

---

The FFmpeg community has some specific guidelines for using Git. It's recommended to read the [git-howto](https://ffmpeg.org/git-howto.html) document. If you're not familiar with Git operations, you can first go through the [Git tutorial by Liao Xuefeng](https://www.liaoxuefeng.com/wiki/896043488029600).

---

The development of features, bug fixes, and discussions about new versions in the FFmpeg community are all very open and transparent. You can follow what all developers are doing through email.

The FFmpeg community uses mailing lists for communication. There are several [Mailing Lists](https://ffmpeg.org/contact.html#MailingLists) available, as shown below:

![1-1](be_contributor\1-1.jpg)

As you can see, there are 5 groups, each dedicated to discussions related to different scenarios or areas.

**1. ffmpeg-devel:** This channel is for discussions related to FFmpeg source code development. New features, functionalities, and codec development are discussed here. By subscribing to this channel, you can see what others are discussing. Yes, anyone can participate in the development and discussions of FFmpeg.

**2. ffmpeg-user:** This channel is for discussions related to the usage of command-line tools like ffmpeg.exe and ffplay.exe.

**3. libav-user:** This channel is for discussions related to using the API functions of the FFmpeg DLL library.

**4. ffmpeg-cvslog:** This channel provides notifications about updates and modifications to the FFmpeg source code. By subscribing to this channel, you will be notified when new features or functionalities are released.

**5. ffmpeg-trac:** This channel is dedicated to reporting bugs.

If you encounter bugs while using FFmpeg, you can search on [ffmpeg-trac](https://ffmpeg.org/pipermail/ffmpeg-trac) as others may have encountered the same issue. This can be faster than using a search engine, as some bugs may not be indexed yet.

---

For those who are not familiar with email communication, the concept of Mailing Lists might be unclear. Mailing Lists are not that different from directly sending or receiving emails.

Email accounts have two recipient roles: **To** and **Cc**.

- **To:** The direct recipient of the email, representing the intended audience. There can be zero to multiple recipients.
- **Cc:** This can be understood as similar to the "@" function on Twitter, used for informing or notifying others. There can be zero to multiple recipients.

The purpose of Mailing Lists is that you can subscribe to a specific email address, essentially subscribing to a channel. For example, with [ffmpeg-trac@avcodec.org](mailto:ffmpeg-trac@avcodec.org), when someone sends an email to this address, a copy will be **Cc'd** to your inbox. Similarly, when you send an email to this address, a copy will be **Cc'd** to the inboxes of other subscribers.

It's recommended to read the [mailing-list-faq](https://ffmpeg.org/mailing-list-faq.html) document.

---

Here's how to subscribe to a `Mailing List`. Open the [ffmpeg-trac subscription page](https://lists.ffmpeg.org/mailman/listinfo/ffmpeg-trac), enter your email address, as shown below:

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3.png" alt="img" style="zoom:50%;" />

If you're a new subscriber and want to see previous discussions, you can find them in the [Archives](https://ffmpeg.org/mailing-list-faq.html#toc-Archives) section.

You can also use Google's site-specific search feature, as follows:

```
site:lists.ffmpeg.org/pipermail/ffmpeg-devel/ "search term"
```

---

FFmpeg also has a [real-time chat room](https://kiwiirc.com/nextclient/irc.libera.chat/?channels=#ffmpeg,#ffmpeg-devel) where you can ask questions and get immediate answers.

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-4-1.png" alt="img" style="zoom:50%;" />

In the image above, the `ffmpeg` channel is for discussions about command-line usage, while the `ffmpeg-devel` channel is for development-related discussions.

---

**Q: How do I apply a patch to my local repository?**

**A:** Use the `git am` command. It's recommended to read the [Fetching Patchwork Patches](https://trac.ffmpeg.org/wiki/FetchingPatchworkPatches) document.

---

**Q: How do I fix a bug reported on FFmpeg?**

**A:** Find the corresponding bug record on [trac.ffmpeg.org](https://trac.ffmpeg.org/). After fixing the bug, package your code changes into a patch file. Then, send the patch via email. If the patch is sent successfully, it will appear on the [patchwork.fmpeg.org](https://patchwork.fmpeg.org/) website.

---

**Q: How do I generate and send a patch via email?**

**A:** Here's an example of how I fixed a very minor issue. I reported a small bug on trac with the ID [9634](https://trac.ffmpeg.org/ticket/9634).

First, execute the following commands to clone the FFmpeg source code and switch to the 4.4 branch:

```
git clone https://git.ffmpeg.org/ffmpeg.git
cd ffmpeg
git checkout remotes/origin/release/4.4
```

I added the following code, which includes a check for `#if CONFIG_AVRESAMPLE`, a macro check that was previously missing:

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-2.png)

Then, execute the following commands to commit the changes to your local repository:

```
git add fftools/cmdutils.c
git commit -m 'fftools/cmdutils.c: add if CONFIG_AVRESAMPLE (ffmpeg version 4.4)'
```

Next, execute the following command to create a patch file from the last commit:

```
git format-patch HEAD^
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-5.png)

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-6.png)

Then, send this patch file to [ffmpeg-devel@ffmpeg.org](mailto:ffmpeg-devel@ffmpeg.org) via email. For guidance on sending emails with Git, you can refer to the article "[How to use git send-email](https://www.jianshu.com/p/fb09b6a87533)" (available in Chinese).

Here's how I configured my Git email settings:

```
[sendemail]
        smtpserver = smtp.qq.com
        smtpuser = loken2022@qq.com
        smtppass = 你的QQ邮箱授权码
        from = 2338195090@qq.com
        to = ffmpeg-devel@ffmpeg.org
        confirm = always
```

Once you have configured the email sending environment, execute the following command:

```
git send-email -1
```

![img](https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-7.png)

After successfully sending the email, you can view the commit record on [patchwork.ffmpeg.org](https://patchwork.fmpeg.org/), as shown below:

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-8.png" alt="img" style="zoom:50%;" />

Since FFmpeg patch submissions are only accepted for the latest commit, and it's February 2022 with the latest commit being for version 5.0, my patch based on 4.4 likely won't get much attention, as the bug was already fixed in later versions.

Because major version updates can lead to API changes or removal, how should bugs in older versions be reported?

For bugs in older versions, you can submit a bug report on [trac.ffmpeg.org](https://trac.ffmpeg.org/) and attach the patch file containing the fixes.

Here's an example of a patch that fixes a [CUDA-related bug](https://trac.ffmpeg.org/ticket/9019):

<img src="https://www.xianwaizhiyin.net/wp-content/uploads/2022/02/contribute-1-3-9.png" alt="img" style="zoom:50%;" />
