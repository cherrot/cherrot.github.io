---
layout: post
title: "硬盘分区变Unknow的修复历程"
date: 2012-11-09 10:24:00
categories: life
tags: partition, ext4
keywords: 
description: 

---

因为时隔久远，具体的修复过程记不清了。只通过IRC日志大概回忆了一下当时的修复过程。一般遇到这种问题原因都是各种各样的，几乎不会有第二个人和你的问题一模一样，所以，实际解决问题的过程中多半靠经验，剩下的就是运气了:)。留下本文，只是想记录一下解决问题的思路而已。

先说一下当时电脑的状况吧。那时还刚来帝都不久，住的地方相当恶劣，不时的还有电脑白痴弄ARP攻击之类。为安全起见，白天上班后就把电脑打开运行[motion摄像头监控](http://linuxtoy.org/archives/linux-camera-monitor.html)。结果不幸发生了，有一天下班回家发现电脑蓝屏，提示BIOS自检出错（Bad checksum）恢复BIOS设定后，发现系统也进不去了，貌似是因为fstab没有办法挂载分区了，最心痛的是，/home也没了……
现在想来可能是夏季电压不稳导致了硬盘故障和系统重启吧，不过也只是事后推测而已。由于事发时我不在现场，导致为什么电脑变成这副德行我都不清楚。

但至少可以确定几点：

1. 我所居住的地方都是电脑白痴，不会有被入侵的可能；
2. fdisk -l能够正确识别分区和其容量，但分区类型是unknow。
3. 我没有进行过与硬盘分区有关的任何操作，所以即使现在看上去再糟糕，应该不会出现数据丢失的状况（坏道除外）；

基于上面的结论，我猜测问题出在分区表上。（后来证明这个猜测是错的，因为分区明明正常，只是系统无法检测分区的文件系统格式而已。这个误判浪费了大量时间……）由于遥远的过去有过一次分区表出错的修复经历，那一次大概是因为用GParted重新调整分区大小时出错导致分区表故障了，用testdisk几乎是傻瓜化地修复了故障。于是这次第一个想到的便是testdisk。

然而这一次testdisk竟然没那么神了，虽然扫到了我的分区，但给出的信息并不正确，抱怨"HDD (500GB/465GiB) seems too small"(还警告 “Incorrect number of heads/cylinder 2 (FAT) != 255(HD)”、“Incorrect number of heads/cylinder 2 (FAT) != 255(HD)Incorrect number of heads/cylinder 2 (FAT) != 255(HD)”之类)，
![testdisk结果](http://t1.qpic.cn/mblogpic/aaebf012367731290d98/2000) 
而且不能够修复（还好testdisk足够强壮没给一个强制修复的选项，不然现在只有哭的份儿了），修改geometry参数重新扫描好几遍都没有结果。难道不是分区表的问题？这下有些慌神了。。。

后来尝试系统自动修复，包括在rocovery下运行fsck，但都没有结果。

后来从IRC里([聊天记录,感激cfy的帮助:)](http://irclogs.ubuntu.com/2012/08/25/%23ubuntu-cn.txt))询问交流了许久，我计划按如下步骤修复硬盘：

1. 使用smartctl或gnome的磁盘工具检查S.M.A.R.T数据有无异常
2. 进入liveCD再想办法。
2. 将/home dd-rescue 到移动硬盘，用testdisk的姊妹程序photorec尝试尽可能的恢复数据
2. 试试patpat等工具
3. 重新分区，尝试使用badblocks修复坏道

第一步中，并没有发现有什么异常数据。进入liveCD后，首先还是寄希望于fsck，或许liveCD下的fsck能给出惊喜。没想到奇迹出现了……执行 fsck.ext4 -pc 后，不一会儿就提示我“one or more group descriptor is invalid”，并询问是否尝试修复错误。选择修复后，伴随着长达半小时甚至更久的跑码，我的分区终于恢复了正常！
分区显示正常后，重启发现仍然不能挂载，有点小奇怪。不过注意到一条提示，大概意思是没能够根据分区的uuid挂载分区。琢磨了一下可能是分区uuid的问题，这个uuid我只记得在分区表定义/etc/fstab和引导配置文件/boot/grub/grub.cfg中存在，于是就检查了一下这两个文件中的uuid，果然有几个分区的uuid和' ls -l /dev/disk/by-uuid/ '给出的结果是不一致的。看来这次蒙对了～ 

更新/etc/fstab后重启，终于又回到了熟悉的Ubuntu！
