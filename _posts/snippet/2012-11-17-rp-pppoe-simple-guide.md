---
layout: post
title: "使用rp-pppoe找回家用路由器账号密码"
date: 2012-11-17 10:24:00
categories: snippet
tags: 
keywords: 
description: 

---

对于一般的路由器来说，最简单的办法就是保存/导出路由器设置，直接用文本编辑器打开
这个文件就能找到。否则就继续看下文。

**首先声明，本文适用于：**

* 找回自己家（自己家的含义在于：你能够物理接触用于拨号的电脑/路由器）的拨号上网密码。

*但不适用于：*

* 破解邻居的无线网络：以我现有的知识，你需要[aircrack-ng](http://www.aircrack-ng.org/)、[reaver-wps](https://code.google.com/p/reaver-wps/) 和一个强大的无线网卡
* 暴力破解路由器登陆密码（非拨号密码）——推荐使用`THC-Hydra`，高效+强大。甚至有图形界面。这里有篇教程：[https://nightx.info/blog/archives/433](https://nightx.info/blog/archives/433)

正文开始。

破解的原理很简单：一般家庭路由器是会轮流尝试不同的认证协议的，而PAP协议需要认证客户端明文发送用户名和密码。基于这两点，我们只需要自己搭建一台只接受PAP认证的pppoe服务器，然后让路由器向自己认证，就可以捕获带有拨号用户名和密码的数据包了。

要点如下：

1. 安装rp-pppoe（在ubuntu下被放到了ppp这一软件包里）
2. 编写服务器配置文件，要点在于require-pap和login：
[https://github.com/cherrot/dotfiles/blob/master/pppoe-server-options](https://github.com/cherrot/dotfiles/blob/master/pppoe-server-options)
3. 运行pppoe-server，一台pppoe认证服务器就运行起来了：
sudo pppoe-server -F -I eth0 -L 100.0.0.1 -R 100.0.0.100 -N 20 -O /PATH/TO/YOUR/pppoe-server-options
4. 启动wireshark，监听以太网接口
5. 拔下路由器WAN口的网线，换成连接自己电脑的网线
6. 找到包含用户名和密码的认证数据帧，任务完成！
