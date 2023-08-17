---
title:  "Proxmark3的重生"
categories:
  - hardware
tags: linux
toc: true
---

最近想要研究一下手上的几张MIFARE卡，就拿出了古早版本的国产PM3。
只是在运行时，它很容易死机，就萌生出了更新固件的念头。

查看了一下购买记录，硬件信息如下：

* 型号: 龙达PM3(512)内存+3.0软件
* 固件: 2017定制版本-suspect 2017-11-11(v3.0)

这款产品目前在商家的店铺还是在售状态，只是固件版本似乎还是3.0，更新自家固件应该是不可行了。

查看了相关帖子，发现可以刷公版的固件，
目前在维护的是[Iceman分支](https://github.com/RfidResearchGroup/proxmark3)。
仔细查看了其介绍，在支持列表中看到它支持众多中国改良版，抱着大不了变砖的心态开始尝试。

我用的是MAC环境，整个过程还是比较流畅、有惊无险，只是有几点需要注意：

* 由于硬件是V3版，因此在安装时要添加`--with-generic`选项，使编译的固件能够兼容我们的硬件。
* 由于bootloader很旧，需要按住按钮刷写，并且需要单独刷写；随后刷写固件。
* 在更新到公版固件后，大部分指令会有变化，也就是说商家配的软件就失效了。

常用的指令可以看[小抄](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/cheatsheet.md)，
试用了最傻瓜的指令：`hf mf autopwn -s 0 -a -k FFFFFFFFFFFF`，
终于不再死机了，效率也很高。

至此，这个闲置多年的设备重获新生。
