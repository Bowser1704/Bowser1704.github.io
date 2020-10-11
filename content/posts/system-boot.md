---
title: "谈谈系统启动发生了什么"
date: 2019-08-22T16:23:18+08:00
draft: false
spotlight: true
categories:
- cs
tags:
- Linux
---

系统启动实际上东西比较多，这里只是讲一讲，系统引导方面的东西。

<!--more-->

### 0. 杂谈

首先我们知道在系统中启动叫做 `boot`，取自 Bootstrap，这里有个小故事

> Bootstrap 不是鞋带的意思，应该是“鞋子背带”的意思（http://en.wiktionary.org/wiki/bootstrap）
> 在这里隐喻表示一种不需要外部帮助自己能够处理事情的情形。“pull oneself up by one's bootstraps”最初来自于《The Surprising Adventures of Baron Munchausen》这本书里的一个故事：主人公 Baron Munchausen 不小心掉进了一片沼泽，他通过自己的 bootstraps 将自己拉了出来（当然有童话神奇的色彩）。事实上在 19 世纪初美国就有"pull oneself over a fence by one's bootstraps"的语言，意思是“做荒谬不可能完成的事情”。
> 参考：http://en.wikipedia.org/wiki/Bootstrapping

为什么说这个故事呢，我们要启动系统，就要启动程序对吧，但是启动程序又要系统，这不就是一个死循环了吗？所以 ` 一种不需要外部帮助自己能够处理事情的情形 `，在当时 ROM（Read only memory）的发展下，人们刚开始发明了 BIOS（Basic Input/Output System），后来又出现了 `UEFI` 全称“统一的可扩展固件接口”(Unified Extensible Firmware Interface)。

### 1. 最开始是如何加载系统的

#### 1.1 老一代的 Legacy BIOS + ``MBR``

最初的启动就是按下电源键之后，电脑读取写入 ROM 的 BIOS。BIOS 开始下面几个步骤。

1. 硬件自检（Power-On Self-Test）简称为 POST，如果有问题计算机会发出不同含义的蜂鸣声。（有没有很傻屌）
2. 选择启动顺序，现在 BIOS 开始要运行的权利给下一个 device 了，但是你电脑可能有多个硬盘，也可能你会插入 U 盘，所以可能需要你选择一个设备。
   - 计算机开始读取设备的第一个扇区，也就是 512 字节，就叫做"主引导记录"（Master boot record，缩写为 ``MBR``）。
   - 但是 `MBR` 有一些问题，所以现在基本不用了。
3. 从硬盘启动，加载系统内核相关的东西。
   - 这里因为 `MBR`，所以系统启动读取的是 `MBR` 内的启动程序，但是如果你一个硬盘，装了很多的系统，每次装新的系统，后面的启动代码就会覆盖前者的。

总结的话 `BIOS` 不认识设备，直接硬的来，你的 `MBR` 内写了什么，他就是什么。

#### 1.2 进入 Linux 系统之后 initd

1. 加载内核，进入 `/boot` 下找到内核文件，加载。
2. 开始运行初始化文件 `/sbin/init`，他的作用是初始化系统环境，`init` 就是第一个进程，`pid`=1
3. 确定运行级别
4. 加载开机启动程序
5. 用户登录
6. 进入 login shell
7. 打开 non-login shell

这里初略讲，详情看阮一峰的文章。

阮一峰写了两篇文章对于以前的启动方式很好的讲解了

http://www.ruanyifeng.com/blog/2013/02/booting.html 参考文章，作者阮一峰，对于 `MBR` 的详细解释

http://www.ruanyifeng.com/blog/2013/08/linux_boot_process.html 参考文章，作者阮一峰，对于 `Linux` 内核启动的解释。

### 2. 今天我们怎么加载系统

#### 2.1 新时代的 `UEFI BIOS`+`GPT`

`UEFI` 不同于 legacy BIOS 的是，他认识设备，他会在 `/boot/efi` 寻找以 `.efi` 为后缀的文件，里面有一个目录 `EFI`，放了各种系统的 `efi` 启动程序。

 例如（不同厂商的驱动放在不同厂家的文件夹下面）

> 启动 windows	进入 Microsoft/Boot/*.efi
>
> 启动 `ubuntu`       进入 ubuntu/*.efi

![efi](https://upload-images.jianshu.io/upload_images/4072641-4b58e0e13c209d14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/345/format/webp)



##### 流程

1. POST，硬件自检

2. `UEFI` 固件加载，一系列初始化。

3. 按照设置里面的顺序，读取 `efi` 启动项，加载硬件驱动，解析其中的分区表（GPT 和 MBR）。

   启动项分为两种

   - 文件启动项，即 `UEFI` 已经记录了启动项的地址，某个磁盘-> 某个分区->/`EFI/Boot/*.efi`
   - 设备启动项，就是磁盘，会直接去磁盘里面寻找 `ESP` 分区下的 `/EFI/Boot/*.efi`

> 随着 Windows8.x，以及 UEFI 标准 2.x，win 推出了一个叫做 SecureBoot 的功能。开了 SecureBoot 之后，主板会验证即将加载的 efi 文件的签名，如果开发者不是受信任的开发者，就会拒绝加载。
> 比如 CloverX64.efi 就好像没有签名。

所以在装 Linux 的时候我们会关闭 SecureBoot，防止不被授权，无法加载 efi

##### 磁盘分区，文件系统

磁盘就是我们用的硬盘，U 盘之类的，我们要对磁盘进行分区，其实不分区可以理解为只分一个区，然后每个区要进行格式化，并且选择文件系统。不同类型磁盘有不同类型的设备驱动，不同系统有不同的文件系统驱动。

- 磁盘驱动
  - `win8/10` 含有 `IDE/SATA/NVME` 三种驱动
  - `macos` 含有 `/SATA/NVME` 驱动，但是 10.13 之前是苹果专用 `NVME` 驱动
- 文件系统驱动
  - `win10` 含有 FAT32、NTFS、exFAT、ReFS 几种
  - Linux 含有 ext2、ext3、ext4、FAT32 等
  - macOS 含有 FAT32、HFS+、APFS、exFAT，NTFS 只读

> 驱动是可以后期安装的，Paragon 这个公司推出了 NTFS for Mac、HFS for Windows、ExtFS for……等一套文件系统驱动。

##### 总结

UEFI 规范里，在 GPT 分区表的基础上，规定了一个 EFI 系统分区（EFI System Partition，ESP），ESP 要格式化成 FAT32，EFI 启动文件要放在“/EFI\< 厂商 >”文件夹下面。**但是 Apple 比较特殊**

作为 UEFI 标准里，钦定的文件系统，FAT32.efi 是每个主板都会带的。所有 UEFI 的主板都认识 FAT32 分区。这就是为啥非得是 FAT32 的。

至于 GPT（GUID Partition Table，缩写：GPT），可以理解为新时代的 `MBR`，分区表。

所以说

>  装系统可以不用制作启动盘了，将 iso 文件压缩到一个 FAT32 分区内，~~ 然后在将该分区下的 `EFI/Boot/BOOTx64` 添加到 UEFI 文件启动项 ~~，直接进入 UEFI 引导，就可以选择这一项启动。

> 另外 UEFI 为了兼容 MBR，是可以用的，但是 Lgacy 不支持 GPT

#### 2.2 Systemd 启动系统

init 启动系统有几点不好

- 串行启动，启动时间长
- 启动脚本复杂

Systemd 是用来替代 init 启动而生的，读作 System daemon（系统，守护进程）。

Systemd 是一组命令，涉及到系统管理的方方面面，本来我们启动系统是用 initd 初始化一个 pid=1 的进程，然后所有进程都是他的子进程，但是现在的话我们不需要，直接启动 Systemd，用它来管理系统。

这个时候就和上面类似了。但是我们要重点关注 `systemd` 命令组。这里就会体现 Linux 下一切皆文件的思想。

![未命名文件](https://ccnupp.oss-cn-shanghai.aliyuncs.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.svg)

#### 这里还要提一点，系统进入之后

首先我们会进入 login shell，这个时候会读取一些配置文件。

```bash
#命令行登录
~/.bash_profile
~/.bash_login
~/.profile	#这里有语句会同时执行.bashrc
#只要读取了上面一个，就不会再读取后面的了，按上述顺序读取。

#图形界面登录的话,只会读取下面两个，不管是否有.bash_profile 存在
etc/ptofile	#这个是对于所有用户的配置文件，~/.bashrc 是对于当前用户的。
~/.profile
```

然后我们手动开启的 shell 叫做 non login shell，他就会读取 .bashrc

### 参考文章

https://www.ibm.com/developerworks/cn/linux/l-lpic1-101-2/index.html	IBM，引导系统

http://www.ruanyifeng.com/blog/2013/02/booting.html 参考文章，作者阮一峰，对于 MBR 的详细解释

http://www.ruanyifeng.com/blog/2013/08/linux_boot_process.html 参考文章，作者阮一峰，对于 Linux 内核启动的解释。

关于 systemd 使用，这里有几篇文章。

http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html	作者：阮一峰

http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html	作者：阮一峰

[https://wiki.archlinux.org/index.php/Systemd_](https://wiki.archlinux.org/index.php/Systemd_(简体中文))	来源 Archwiki






