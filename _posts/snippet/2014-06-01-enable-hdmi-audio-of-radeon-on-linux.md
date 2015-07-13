---
layout: post
title: 为老旧Radeon显卡启用HDMI音频
date: 2014-06-01 23:02:49
categories: snippet
tags: linux hdmi radeon audio
description: 在Linux环境下为老旧Radeon显卡型号启用实验性HDMI音频支持

---
工作了愈发体会到眼睛的重要。学生时代买的千元级IPS LED屏幕，一直觉得眼睛不舒服，而且时不时会注意到屏幕有频闪。当时还没太在意，乱猜这只是电流不稳导致。因为印象中以为到了液晶时代不会再有频闪的显示器了。
 
但因为最近在处理照片，和写代码不同，处理照片时眼睛是高度集中在屏幕上的，就像玩游戏时一样，这时候眼睛相当难受，甚至会不停流眼泪出来。我觉得必须要重视一下这个问题了。知乎上看到[这个问题][ref-zhihu]后我才知道我对LED显示屏的无知导致我的眼睛忍受了三年的摧残。本着健康第一的原则，海淘了一副Gunnar眼镜在公司戴，网购了一台BenQEW2440替掉之前的飞利浦劣质屏。这个屏只有VGA和HDMI接口，还好我的Raedon 4200 HD的板载显卡(主板是映泰TA880G HD)支持HDMI。不过问题来了：我的Ubuntu 13.10 高清显示毫无压力，但是HDMI里没有声音输出。
 
Google之，有说启用 S/PDIF 数字音频输出就可以解决问题，但在我的板子上 S/PDIF 应该是专用的数字音频输出口，`aplay -l` 中可以看到如下输出：
 
```text
card 0: SB [HDA ATI SB], device 0: ALC892 Analog [ALC892 Analog]
子设备: 1/1
子设备 #0: subdevice #0
card 0: SB [HDA ATI SB], device 1: ALC892 Digital [ALC892 Digital]
子设备: 1/1
子设备 #0: subdevice #0
card 1: HDMI [HDA ATI HDMI], device 3: HDMI 0 [HDMI 0]
子设备: 0/1
子设备 #0: subdevice #0
```
 
继续Google, 终于找到了[答案][ref-radeon]。其实开源显卡驱动是实验性支持Radeon早期系列显卡的HDMI音频的，但因为在个别产品上会出现白屏等稳定性问题，所以默认禁用了。既然找到了问题根源，那解决起来也就可以对症下药了。
 
##在Radeon系列显卡上启用开源视频驱动对HDMI音频的支持
 
要想启用HDMI音频支持，需要添加内核参数`radeon.audio=1`，比较优雅的方式如下（假定你使用Grub2作为默认启动管理器）：
 
1. 编辑`/etc/default/grub`文件，找到`GRUB_CMDLINE_LINUX_DEFAULT`所在的行，在最后添加`radeon.audio=1`，在我的机器上添加后是这样子：
 
    ```linux-config
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.audio=1"
    ```
 
2. 运行 `sudo update-grub` 更新`grub.cfg`
 
等到更新完成后，重启系统，这时候打开系统的声音设置面板，就会发现多出了`HDMI Audio`的选项了，切换到这个音频输出设备就可以了，祝好运! ;)
 
##已知问题
 
虽然我没有遇到白屏或者其他严重的显示问题。但实测发现在持续的声音输出时HDMI音频会有持续的杂音。
 
##解决杂音
 
感谢万能的 [ArchWiki][ref-archwiki-pulseaudio], 解决方法是编辑`/etc/pulse/default.pa`,添加参数`tsched=0`:
 
```text
/etc/pulse/default.pa
load-module module-udev-detect tsched=0
```
 
然后重启 pulseAudio:
 
```bash
pulseaudio -k
pulseaudio --start
```
 
更多关于HDMI音频输出问题的排查，猛戳[ArchWiki][ref-archwiki-alsa]。
 
##参考链接
 
1. [知乎 - 明基不闪屏显示器真对眼睛有好处吗？怎么实现的？][ref-zhihu]
2. [help.ubuntu.com - RadeonDriver][ref-radeon]
3. [Audio over HDMI and DisplayPort in Ubuntu 12.04][ref-hdmi-audio]
4. [Ask Ubuntu - How do I add a kernel boot parameter?][ref-kernel-param]
 
 
[ref-zhihu]: http://www.zhihu.com/question/21127560 "知乎 - 明基不闪屏显示器真对眼睛有好处吗？怎么实现的？"
[ref-radeon]: https://help.ubuntu.com/community/RadeonDriver#HDMI_Audio "help.ubuntu.com - RadeonDriver"
[ref-hdmi-audio]: http://voices.canonical.com/david.henningsson/2012/04/14/audio-over-hdmi-and-displayport-in-ubuntu-12-04/ "Audio over HDMI and DisplayPort in Ubuntu 12.04"
[ref-kernel-param]: http://askubuntu.com/questions/19486/how-do-i-add-a-kernel-boot-parameter "Ask Ubuntu - How do I add a kernel boot parameter?"
[ref-archwiki-pulseaudio]:https://wiki.archlinux.org/index.php/PulseAudio#Glitches.2C_skips_or_crackling "PulseAudio"
[ref-archwiki-alsa]: https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture#HDMI_output_does_not_work "ALSA wiki"
