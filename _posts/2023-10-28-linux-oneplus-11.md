---
title:  "一加11(PHB110)刷机和Root指南"
last_modified_at: 2023-12-05T00:00:00+00:00
categories:
  - hardware
tags: linux android
toc: true
---

在搭载高通8GEN2处理器的一系列手机中，一加11凭借泄露的9008工具，一举成为了最受开发者关注的机型。
截至今日，该型号已适配第4个第三方ROM，且在XDA Forums中的讨论度排行靠前。
一切迹象表明，是时候更新手中的开发机了。

## 刷机工具

比较遗憾的是，大部分刷机工具只能运行在Windows环境下。

### Latest ADB Fastboot and USB driver installer tool

用于一键安装ADB和Fastboot模式下的驱动

地址：[github](https://github.com/fawazahmed0/Latest-adb-fastboot-installer-for-windows)

### 欧加真9008工具

基于[bff](https://gitee.com/mouzei/bff)，用于备份和还原Flash文件

作者：[某贼](https://www.coolapk.com/feed/49808862?shareKey=Nzk3ZjNmN2I2NWNjNjUzZDI0MjQ~)

地址：[lanzou(f65u)](https://syxz.lanzoub.com/b01fiq7sb)

### OPLUS骁龙8GEN2设备刷机工具

用于PHB110机型分区的9008刷入

作者：[秋水105](https://www.coolapk.com/feed/48761687?shareKey=ZDQwMDk0Zjc3NTNhNjUzZDI2ZjI~)
[更新1.3](https://www.coolapk.com/feed/50377218?shareKey=YWMyNmJkYTFkODdhNjUzZDIyNzA~)

地址：[cyteam](https://pan.cyteam.cn/s/5rXib)

### FastbootEnhance

提供Fastboot的GUI操作，包括切换到fastbootd等。

地址：[github](https://github.com/libxzr/FastbootEnhance)

## 备份分区

为了避免参数丢失不得不换主板的悲剧，在进行任何操作前必须备份全部分区。
目前，可使用**欧加真9008工具**一键备份（需要安装9008驱动）。

特别的，在进行root时需要对boot分区进行修补，故需要频繁对boot分区进行备份。
通过比较OTA和OFP包可发现，同样的版本下，不同打包方式中的boot镜像是相同的。
另外，在未修改boot分区时，备份得到的镜像与刷机包中的一致。

## 刷入OOS

由于COS和OOS中比较大的分区变动[^JimmyTian1]，刷机过程比较复杂，主要流程如下[^jarodlau1]：

解锁bootloader：

1. 打开developer mode 开发者模式
2. 打开USB Debug usb调试
3. 开启 OEM-unlock oem解锁
4. adb reboot-bootloader
5. 进入 fastboot
6. fastboot flashing unlock
7. adb reboot-bootloader

刷入系统：

1. 下载CPH2449 GLO A.10
2. 打开Fastboot Enhance
3. 确认勾选"Ignore Unknown Partition"
4. 确认 fastbootd is NO
5. Flash payload.bin
6. 选择 boot into fastbootd
7. 打开 Fastboot Enhance
8. 再三确认"Ignore Unknown Partition"
9. 确认 fastbootd is YES
10. Flash payload.bin
11. 下载解压 ocdt_CPH2449.img
12. 进入fastboot
13. fastboot flash ocdt ocdt_CPH2449.img
14. Fastboot reboot

回锁bootloader：

这里要注意不能在初始化阶段设置密码[^daxiaamu1]，并且不要在修改boot分区后回锁[^daxiaamu2]。

1. 进入OxygenOS, 跳过所有选项. 不要设置pin密码（此过程最好不要插卡）。 🈶科学环境的话，无所谓。
2. 打开 developer mode开发者模式
3. 打开 USB Debug usb调试
4. 打开OEM-unlock
5. adb reboot-bootloader
6. 进入fastboot模式
7. fastboot flashing lock
8. fastboot reboot

经测试，在回锁后，系统能够正常增加更新。（在Root后无法OTA更新[^community1]）

刷入my_preload分区：

由于该分区存放的是预装应用等内容，不需要在OTA中更新，故在上述刷写过程中不会被覆盖[^c540690p1]。
因此，需要从**OFP**包中提取并刷入[^jarodlau2]

## root(KernelSU)

[KernelSU](https://github.com/tiann/KernelSU)是一种新的root方式（对内核进行修改），隐蔽性更高，但操作也更复杂。
对于一加11的原版ROM，目前均已有现成的内核修补包供直接刷入（不压缩）；
但是如果刷了第三方系统，则可能需要手动编译内核。

在刷入前，可以通过如下指令尝试启动：

```shell
fastboot boot boot.img
```

确认能够启动后，再实际刷入：

```shell
fastboot flash boot boot.img
```

KernelSU在模块接口方面尽可能地与[Magisk](https://github.com/topjohnwu/Magisk)保持一致，并提供了额外的功能。
另外，对于Magisk的**Zygisk**功能，通过[Zygisk Next](https://github.com/Dr-TSNG/ZygiskNext)模块实现了支持。

在**需要**时可以给**shell**添加root，这样就能通过`adb shell`执行root脚本。

## Xposed框架(LSPosed)

[LSPosed](https://github.com/LSPosed/LSPosed.github.io)是一个基于Xposed框架的[^xposed]安卓应用hook模块，目前已支持到安卓14。
LSPosed虽然以模块的形式刷入，但自身又是一个强大的框架，搭载了大量的Xposed模块。
通过Xposed框架定义的一套hook接口，Xposed模块的开发者可以不需要适配安卓底层的改动，应用的兼容性得到显著提升。

### CallRecording

该[应用](https://github.com/vvb2060/CallRecording)解决了原生拨号器无法录音的问题：
1. 将`CountryCode`修改为可录音的区域。
2. 跳过开启录音时的提示音（根据拨号器版本分为2种方式）

### SSLUnpinning

随着安卓的更新，对应用的抓包（MITM攻击）变得越来越困难，这一方面体现在证书的安装上[^mitmproxy1]，另一方面体现在应用的自保护上。
[SSLUnpinning](https://github.com/Xposed-Modules-Repo/io.github.tehcneko.sslunpinning)通过绕过应用的CA证书有效性检测的方式，实现了对大部分应用的抓包。

## ROM提取

卡刷（OTA）包是单个bin文件，需要先将其解包，可使用[payload-dumper-go](https://github.com/ssut/payload-dumper-go)。

线刷（OFP）包是按分区分割的img文件，格式为**Android sparse image**，需先使用[simg2img](https://github.com/anestisb/android-simg2img)将其转换为正常的img文件，然后挂载到linux系统中读取。

一加的内置系统应用在分区`my_stock`中。
其中，`我的一加`软件在文件夹`KeKeUserCenter`中，在COS中包名为`com.heytap.vip`，在OOS中包名为`com.oneplus.account`，两者并不能覆盖安装。
其它的一加服务都依赖于上述软件，因此提取系统应用并没有什么意义。

## Reference

[^JimmyTian1]: 或许是色刷氧后奇怪问题的解决方法, [url](https://www.coolapk.com/feed/45800982?shareKey=MGZkZjgxZjQ3NjVhNjUzZDI4ZmE~)

[^jarodlau1]: 应该是目前为止最完美的刷入, [url](https://www.coolapk.com/feed/47110139?shareKey=ZjdjNWE0ODNmZTRhNjUzZDI4ZDI~)

[^daxiaamu1]: 一加11解锁后无法设置锁屏密码的处理方法：100%成功, [url](https://www.daxiaamu.com/7601)

[^daxiaamu2]: ROOT后上锁无法重新解锁, [url](https://www.daxiaamu.com/7694)

[^c540690p1]: 关于为什么刷完氧OS13还是有一堆内置国行app这件事, [url](https://www.coolapk.com/feed/43883313?shareKey=ZWY5ZTBmZWUxMmFlNjUzZDJjOTE~)

[^jarodlau2]: 一加11 出厂cos完美刷入oos的教程, [url](https://www.coolapk.com/feed/43732413?shareKey=YTY4ZjExMTRjZjA2NjUzZDJkYjk~)

[^community1]: All about OxygenOS - September 2023, [url](https://community.oneplus.com/thread/1428579350231384065)

[^xposed]: Xposed - General info, versions & changelog, [url](https://xdaforums.com/t/xposed-general-info-versions-changelog.2714053/)

[^mitmproxy1]: Install System CA Certificate on Android Emulator, [url](https://docs.mitmproxy.org/stable/howto-install-system-trusted-ca-android/)
