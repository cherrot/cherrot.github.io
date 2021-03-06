---
layout: post
title: Linux下安装Win7二三事
date: 2013-09-20 23:02:49
categories: snippet
tags: windows
description: 

---

参考[知乎原文](http://zhi.hu/VW0c "怎样安装 Windows 7 与 Linux 共存的双系统（在linux系统下）？") 如果已经安装了Linux，要在此基础上安装Win7的话，事情可能没有想象的那么简单。总结一句话：Windows一次次挑战我的极限。亲自试验可行的安装方法见**第二部分**。注意，本文仅限于使用安装版光盘镜像安装Windows系统，Ghost方式请另行搜索。 

## 第一部分：使用ms-sys引导U盘安装

首先，直接`dd if=win7.iso of=/dev/sdb`是不行的， 其次，`grub2`的`loopback`功能对Win7的ISO不起作用（`loopback`只支持iso9660格式的光盘镜像），于是没有办法像linux启动盘那样配置一下`grub.cfg`，放个linux的ISO镜像那么简单明快。 最简单的解决方案见Ubuntu中文论坛的[linux中制作win7安装U盘](http://forum.ubuntu.org.cn/viewtopic.php?f=77&t=385738) ：

> 需要gparted、ms-sys、win7安装用ISO，4G及以上大小U盘一个
> 
> 1.  用gparted在U盘上建立一个ntfs分区，编辑flag，勾上boot选项。 这个分区是用来存放win7iso的内容的，所以大小一定要够大，boot选项也就是设为活动分区的意思。
> 2.  挂载win7.iso和新建的ntfs分区，并将全部内容复制到那个ntfs分区。
> 3.  编译安装ms-sys。ms-sys是一个写mbr的工具，起到让系统知道能够引导win7安装的作用，至关重要：<http://ms-sys.sourceforge.net> 安装就是make，会在代码目录下生成 bin/目录, cd 到 bin/目录下
> 4.  运行：
>    ``` shell
>    ms-sys -7 /dev/sdX
>    ```
>     写入mbr。其中的-7参数指win7, sdX指的是U盘对应的盘符（一般为 sdb）
> 
> 这样就搞定了，注意XP的安装盘是不可以这么做的。

重启，U盘启动，安装系统（需要事先保证有或能有空闲的**主分区**用于安装Win7）即可。 但是很遗憾，使用U盘安装过程中我遇到“安装程序无法定位现有系统分区,也无法创建新的系统分区” 错误。我曾以为是分区格式问题，重新格式化也是徒劳。而且安装程序中对分区采取的操作不会向我确认就会被执行（血的教训！一个简单的删除分区操作竟然导致分区表错位，fdisk重建分区才得以修复）。 总之，这种安装方式不可行（原因可见下文总结）。 

## 第二部分：使用Grub引导安装

知道上述错误失败的原因后，我便尝试使用`Grub`的`bootloader`引导Windows安装程序完成系统安装。为了保险，这次没有使用U盘，而是将windows安装盘镜像解压到了硬盘的一块ntfs分区上（我没有测试使用这种引导方法引导U盘安装程序能否解决第一部分中出现的问题）。 首先在硬盘或者U盘等可启动介质上安装`Grub`：

``` shell
sudo grub-install --root-directory=/media/cherrot/myUDisk /dev/sdb
```

然后，在引导目录（此例中为`/media/cherrot/myUdisk**/boot/grub/**`）中创建`grub.cfg`文件，内容可参考如下配置(关于Grub，[GRUB - ArchWiki](https://wiki.archlinux.org/index.php/GRUB) 是个好去处。配置中两个Win7的菜单项效果等同，注意修改配置中的加粗的分区号和uuid)：

``` conf
set root='(hd0,1)'
set timeout=10
#grub2 loopback does not support Win7.iso, because it's not iso9660 format.
menuentry "WTF Win7" {
    #I have put the install file in the 6th partition
    set root='(hd0,6)'
    #chainloader /bootmgr
    chainloader +1
}
#GRUB supports booting bootmgr directly and chainload of partition boot sector is no longer required to boot Windows in a BIOS-MBR setup.
menuentry "Microsoft Windows Vista/7/8 BIOS-MBR" {
  insmod part_msdos
  insmod ntfs
  insmod search_fs_uuid
  insmod ntldr     
  search --fs-uuid --set=root --hint-bios=hd0,msdos6 --hint-efi=hd0,msdos6 --hint-baremetal=ahci0,msdos669B235F6749E84CE
  ntldr /bootmgr
}

#Following configs show how easy it is to install Linux.
menuentry "Ubuntu 13.04 Gnome Desktop AMD64" {
    loopback loop (hd0,1)/boot/iso/ubuntu-gnome-13.04-desktop-amd64.iso
    linux (loop)/casper/vmlinuz.efi boot=casper iso-scan/filename=/boot/iso/ubuntu-gnome-13.04-desktop-amd64.iso
    initrd (loop)/casper/initrd.lz
}

menuentry "Ubuntu 12.04 Alternate AMD64" {
    loopback loop (hd0,1)/boot/iso/ubuntu-12.04-alternate-amd64.iso
    linux (loop)/install/vmlinuz boot=install iso-scan/filename=/boot/iso/ubuntu-12.04-alternate-amd64.iso
    initrd (loop)/install/initrd.gz
}
```

好了，重启，使用写入grub的引导介质启动，引导进入Windows安装程序，一路快车完成安装！ 

## 总结

如果已有一个Windows或Windows引导区的话，可以参考 [How to install and boot Windows on a Logical Parition](http://www.zyxware.com/articles/2008/07/20/how-to-install-and-boot-windows-on-a-logical-parition) 总之至少要保证有一个主分区存放windows启动文件。 

一个简单的系统安装，竟然都要经历这样的波折，这让我感到非常遗憾。一个好的安装程序绝不会隐藏太多的细节让人觉得它在操纵一个黑盒子，甚至出现问题都不能知道原因（这一点最让我不能忍受，因为把出错信息贴到google后我找了三天才从一堆**毫无建设意见的“解决方法”帖**中找到一条链接 [安装Windows出现“安装程序无法创建新的系统分区，也无法定位现有系统分区”](http://bbs.itmop.com/thread-46041-1-1.html) 告诉了我为什么安装程序会抱怨无法定位分区：使用U盘安装win7的时候，加载到安装程序中的u盘变成了主引导盘）。 上文链接中作者的吐槽我深感赞同：

> Looking at all this pain, I wonder how easy it is to set up GNU/Linux on any partition and the ease of configuration of the boot files and their parameters. When everything works fine both Windows and GNU/Linux are comparable but when things go wrong, they both require almost similar levels of expertise to fix. I don't know who spread the myth that Windows is easier to configure. Probably those people who never had to configure anything beyond few radio buttons/check boxes or few text fields. Hard core users would find both equally good, equally involved in configuring and probably equally challenging. However being fully open GNU/Linux offers far more flexibility and control in configuration than Windows.

P.S. 从一个U盘安装Win7的帖子中找到了灵感，在使用U盘安装windows时，如果把当前硬盘的MBR引导区写入U盘，或许就可以解决无法定位分区的问题了。尚未验证，但应该没问题 :)
