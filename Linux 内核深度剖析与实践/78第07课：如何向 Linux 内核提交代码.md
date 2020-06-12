# 7/8第07课：如何向 Linux 内核提交代码

现在我们有了内核的相关理解，如何向内核提交自己的代码呢？这是一个漫长的道路，首先你要知道内核的各种使用，这一点希望在工作中大家能随经验的积累对内核有深入的理解，这里给大家分享更多的是在术的方面如何向内核提交代码，有哪些步骤走。

### 准备工作

工欲善其事，必先利其器。进入正题之前先准备下需要安装和配置的环境和工具。首先要安装 Linux 操作系统，有很多发行版，如 Ubuntu、Centos，看个人兴趣去选择，本人比较喜欢 Ubuntu，这里主要以 Ubuntu 操作系统为例，给大家演示准备工作。

#### 安装和配置 msmtp

点击左边栏的“软件中心”，在搜索框中输入“msmtp”，选择安装即可，然后按照如下步骤配置 msmtp：

![image](http://images.gitbook.cn/a936f990-e6de-11e7-a892-07c9a72d7510)

#### 安装和配置 Git 环境

默认的 Linux 系统一般都已经安装好 Git。如果没有，随便找一本 Git 的书都可以，这里不详述。比较好的 Git 资料[请参考这里](http://git.oschina.net/progit/)。

在配置用户名的时候，请注意社区朋友习惯用英语沟通，也就是名在前，姓在后。这一点会影响社区邮件讨论，因此需要留意。在配置邮箱时，也要注意。社区会将国内某些著名的邮件服务器屏蔽。因此建议你申请一个 Gmail 邮箱。举例配置如下。

（1）订阅 mailinglist

Linux 开源分支要求开发者上传 patch 或者 driver 时，需要将邮件抄送给 mailinglist、maintainer 和其他人，首先需要订阅相关子系统的 mailinglist。

订阅邮件列表 mailing list：//改成自己所要提交代码所在子系统的 mailing list，详见 Linux 代码根目录下的 MAINTAINERS 文档：

![image](http://images.gitbook.cn/013d2790-e6df-11e7-a892-07c9a72d7510)

注意：订阅 mailinglist 的邮件不需要标题，请参考如下方式：

![image](http://images.gitbook.cn/1c5ef490-e6df-11e7-9926-cbffa6ccfcea)

注意：发送完 subscribe 邮件后，你会收到一封确认邮件，比如我的确认邮件标题为“Confirmationfor subscribe linux-media”，里面有认证信息，请按照邮件内容，再发一个认证邮件给 majordomo@vger.kernel.org，如下：

![image](http://images.gitbook.cn/35c79fe0-e6df-11e7-a892-07c9a72d7510)

（2）下载 Linux 源代码

首先打开以下网页选取想要工作的分支：[https://www.kernel.org](https://www.kernel.org/)。

![image](http://images.gitbook.cn/64513ab0-e6df-11e7-b999-2702308e606e)

从下载的代码里选取感兴趣的模块，你可以在内核源码目录 \MAINTAINERS 文件中，找一下相应文件的维护者，及其 Git 地址。例如，watchdog 模块的信息如下：

![image](http://images.gitbook.cn/7f9360f0-e6df-11e7-9926-cbffa6ccfcea)

其中，[单击这里获取 Git 地址](git://www.linux-watchdog.org/linux-watchdog.git)，可以用如下命令拉取 watchdog 代码到本地：

![image](http://images.gitbook.cn/a09b0960-e6df-11e7-963d-d9deae93df7c)

当然，这里友情提醒一下，MAINTAINERS 里面的信息可能不一定准确，这时候你可能需要借助 Google，或者问一下社区的朋友，或者直接问一下作者本人。不过，一般情况下，基于 linux-next 分支工作不会有太大的问题。实在有问题再去打扰作者本人。

（3）阅读 Documentation/SubmittingPatches，这很重要。

### 制作补丁

#### 补丁描述

补丁第一行是标题，比较重要。它首先应当是模块名称。

举个例子，怎么找到 drivers/clk/samsung/clk-s3c2412.c 文件属于哪个模块？可以试试下面这个命令，看看 drivers/clk/samsung/clk-s3c2412.c 文件的历史补丁：

![image](http://images.gitbook.cn/e8af7d80-e6df-11e7-9464-f59d5bc661bc)

可以看出模块名称是 “clk:samsung”。下面是我为这个补丁添加的描述，其中第一行是标题：

```
clk: samsung: mark symbols static where possible for s3c2410 
We get 1 warnings when building kernel withW=1:
/dimsum/git/kernel.next/drivers/clk/samsung/clk-s3c2410.c:363:13:warning: no previous prototype for 's3c2410_common_clk_init'[-Wmissing-prototypes]
 void __init s3c2410_common_clk_init(struct device_node *np, unsigned long xti_f,

In fact, this function is only used in thefile in which they are declared and don't need a declaration, butcan be made static.
So this patch marks these functions with 'static'.
```

这段描述是我从其他补丁中复制出来的，有几下几点需要注意：首先标题中故意添加了“for s3c2410”，以区别于另外两个补丁。其次“1 warnings”这个单词中，错误的使用了复数，这是因为复制的原因。

接着“/dimsum/git/kernel.next/”这个路径名与我的本地路径相关，不应当出现在补丁中。最后警告描述超过了80个字符，但是这是一个特例，这里允许超过80字符。

这些问题，如果不处理的话，Maintainer 会不高兴的！如果 Maintainer 表示了不满，而你不修正的话，这个补丁就会被忽略。修正后的补丁描述如下：

```
clk: samsung: mark symbols static wherepossible for s3c2410
We get 1 warning when building kernel withW=1:
drivers/clk/samsung/clk-s3c2410.c:363:13:warning: no previous prototype for 's3c2410_common_clk_init'[-Wmissing-prototypes]
void__init s3c2410_common_clk_init(struct device_node *np, unsigned long xti_f,
In fact, this function is only used in thefile in which they are declared and don't need a declaration, but can be made static.
So this patch marks these functions with 'static'.
```

我们的补丁描述一定要注意用词，不要出现将“unused”写为“no used”这样的错误。反复使用 git add，git commit 将补丁提交到 git 仓库。

#### 如何生成补丁

有很多的场景根据不同需求生成补丁，这里介绍两种工作中常用遇到的场景：

```
# git format-patch HEAD^
0001-au0828-fix-logic-of-tuner-disconnection.patch
# cat 0001-au0828-fix-logic-of-tuner-disconnection.patch
From cc4f6646ae5eb0d75d56cca62e2d60c1ac8cad66 Mon Sep 17 00:00:00 2001
From: Changbing Xiong <cb.xiong@samsung.com>
Date: Tue, 22 Apr 2014 16:10:29 +0800
Subject: [PATCH] au0828: fix logic of tuner disconnection //此处的 [PATCH] 是工具自动加上的

The driver crashed when the tuner was disconnected while restart stream
。。。。。。。
restart stream operations has been released.

Change-Id: Iaa1b93f4d5b08652921069182cdd682aba151dbf //需要通过 vim 删除此行
Signed-off-by: peter liu <peter.liu@gitchat.com>
---
 drivers/media/usb/au0828/au0828-dvb.c |   13 +++++++++++++
。。。。。。。
```

上面是将最近一次的修改生成一个 patch，不过注意，如果 patch 中有 Change-Id 行，需要删除。

#### 检查补丁

在发送补丁前，我们需要用脚本检查一下补丁：

```
./scripts/checkpatch.pl 000*
---------------------------------------
0001-clk-samsung-mark-symbols-static-where-possible-for-s.patch
---------------------------------------
WARNING: Possible unwrapped commit description (prefer a maximum 75 chars per line)
#9:
 void__init s3c2410_common_clk_init(struct device_node *np, unsigned long xti_f,

WARNING: line over 80 characters
#29: FILE:drivers/clk/samsung/clk-s3c2410.c:363:
+static void __init s3c2410_common_clk_init(struct device_node *np, unsigned long xti_f,

total: 0 errors, 2 warnings, 8 lineschecked 
```

请留意输出警告，其中第一个警告是说我们的描述中，有过长的语句。前面已经提到，这个警告可以忽略。但是第二个警告告诉我们代码行超过80个字符了。这是不能忽略的警告，必须处理。

#### 补丁的提交

在第一次 commit 时使用 -s，后面修改下面内容时，用 —amend 即可。

```
au0828: fix logic of tuner disconnection                     //标题，带上模块名称，如au0828
                                                                                             //此处必须空一行
The driver crashed when the tuner was disconnected while restart stream
operations are still being performed. Fixed by adding a flag in struct
au0828_dvb to indicate whether restart stream operations can be performed.

If the stream gets misaligned, the work of restart stream operations are
 usually scheduled for many times in a row. If tuner is disconnected at
this time and some of restart stream operations are still not flushed,
then the driver crashed due to accessing the resource which used in
restart stream operations has been released.
                                                                                             //此处必须空一行
Signed-off-by: Changbing Xiong <cb.xiong@samsung.com>    //通过git commit –s 自动生成

# Please enter the commit message for your changes. Lines starting  //从这往下都是自动生成，勿动
# with '#' will be ignored, and an empty message aborts the commit.
# On branch tizen
# Your branch is ahead of 'origin/tizen' by 1 commit.
#
# Changes to be committed:
#   (use "git reset HEAD^1 <file>..." to unstage)
#
# modified:   drivers/media/usb/au0828/au0828-dvb.c
# modified:   drivers/media/usb/au0828/au0828.h
```

#### 发送补丁

生成正确的补丁后，请再次用 checkpatch.pl 检查补丁正确性。确保无误后，可以准备将它发送给 Maintainer 了。但是应该将补丁发给谁？这可以用 get_maintainer.pl 来查看。接下来，可以用 git send-email 命令发送补丁了：

```
git send-email 000* --tokgene@kernel.org,krzk@kernel.org,s.nawrocki@samsung.com,tomasz.figa@gmail.com,cw00.choi@samsung.com,mturquette@baylibre.com,sboyd@codeaurora.org--cc linux-arm-kernel@lists.infradead.org,linux-samsung-soc@vger.kernel.org,linux-clk@vger.kernel.org,linux-kernel@vger.kernel.org
```

注意哪些人应当作为邮件接收者，哪些人应当作为抄送者。在本例中，补丁是属于实验性质的，可以不抄送给邮件列表帐户。提醒：读者应当将补丁先发给自己，检查无误后再发出去。如果有朋友在社区有较高的威望，也可以抄送给他，必要的时候，也许他能给一些帮助。这有助于将补丁顺利的合入社区。重要提醒：本文讲述的，主要是实验性质的补丁，用于打开社区大门。真正重要的补丁，可能需要经过反复修改，才能合入社区。

### 最后

至此，本系列达人课暂告一段落，感谢 GitChat 平台，感谢读者朋友们，希望此系列课程对大家有所帮助，明年我们一起走进新的嵌入式达人课系列。