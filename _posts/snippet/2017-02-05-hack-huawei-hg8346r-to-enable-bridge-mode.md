---
layout: post
title: 破解华为HG8346R光猫 启用有线桥接
date: 2017-02-05 18:58:00
categories: snippet
tags: 路由器
description: 华为`HG8346R`光纤路由器自带的功能限制太多，而且无线路由能力很弱，破解后可以开启有线桥接模式（交换模式），使用单配的无线路由器拨号连接，最大化发挥网络性能。本文适用于OSX, Windows 和 Linux系统。

---

## 为何破解

华为这款光猫+无线路由器一体机性价比还是挺高的，普通上网需求足够了。但是一年时间用下来，我发现这台设备有不少问题：

1.  无线很不稳定，接入设备多或者网络流量大时，无线经常掉线重连。这一点在知乎也有人验证了，可能和光猫本身的CPU处理能力有关。
2.  无线实时性差，带宽也不高，不支持5GHz频段。虽然理论能到300Mbps，但实际差得远呢。
3.  联通定制版限制太多，很多设置作为普通用户无法修改，甚至连无线SSID都强制以`CU_`开头，日了狗。

我的所有联网设备--手机，摄像头，iPad，笔记本甚至台式机--都使用的无线接入，这么下去肯定不行。所以计划单独买一个无线路由器提高无线性能，同时让光猫单纯作为一台交换机，这样也就避免了光猫CPU性能不足导致的掉线、丢包等问题了。不过定制版无法开启WAN口桥接，只好参考[别人的破解步骤](http://post.smzdm.com/p/441624/)来一发了。

多解释一句：WAN口桥接模式，其实就是将光猫作为交换机使用，转发二层数据帧可比路由三层IP包快多了，否则光猫的CPU依然会成为瓶颈。

## 准备工作

本文适用于（北京）联通定制版的华为`HG8346R`光纤接入终端，软件版本（可以在web管理界面找到）：`V3R015C10S109`。

另外还需要下载华为ONT维护工具包用于开启维护模式和加解密配置文件，以及tftp服务器用于备份路由器配置。可以在我的[Google Drive][google-drive]或[Mega网盘][mega]下载。

### OSX / Linux

华为的维护工具包是Windows程序，因此需要安装`wine`模拟器：

``` bash
# on OSX
brew install wine
# on Ubuntu Linux
sudo apt-get install wine
```

不过tftp服务器不能直接用`wine`模拟，所以需要单独安装。建议使用[Python的开源实现](https://wiki.python.org/moin/tftp)，我用的[TFTPy](http://tftpy.sourceforge.net/sphinx/index.html)：

``` bash
sudo pip install TFTPy
```

只要三行代码就能启动一个TFTP服务器了：

``` python
import tftpy

server = tftpy.TftpServer('./tftp-dir')
server.listen('0.0.0.0', 69)
```

或者`cd`到工具包目录，`sudo python tftpd.py`启动tftp服务

注意，OSX用户**不要使用系统自带的tftpd服务，基于tftpd的GUI版软件也不行（比如tftpserver）**，因为要想上传文件（`put`），必须更改tftpd的启动参数（`-s`），但`OS X EI Capitan`禁用了编辑服务配置文件的权限，`sudo`也没戏，除非重启进入Recovery模式。另外我尝试直接运行`tftpd`也没戏，折腾个把小时还不如3行Python代码来的简单。

### Windows

工具包里带了`tftpd32`和`tftpd64`程序，直接运行就可以了。

## Let's hack!

### 备份配置

我们假定电脑的IP是`192.168.1.2`，路由器IP为`192.168.1.1`。建议直接设置为静态IP。

1.  确保光猫开机，光纤拔出，电脑用网线连到光猫LAN1口。
2.  运行`华为ONT组播版本配置工具.exe`（OSX执行`wine 华为ONT组播版本配置工具.exe`，下文不再赘述）。  
    选择正确的网卡（IP为`192.168.1.2`），其他保持默认，点击`启动`按钮。
3.  开始时光猫的光信号灯红色闪烁（表示未接入光纤），耐心等待2分钟左右，如果所有灯绿色常亮了，说明成功开启了维护模式，可以`停止`并关闭工具了。
4.  重启光猫，打开终端或命令行：

    ``` bash
    telnet 192.168.1.1

    root #（Login用户名）
    admin #（密码）
    ```
5.  确保tftp服务正常，在`telnet`中执行：

    ``` bash
    backup cfg by tftp svrip 192.168.1.2 remotefile hw_ctree.xml
    ```
    如果命令执行成功，你设定的tftp目录下（tftp-dir）便有了`hw_ctree.xml`配置文件了。
6.  运行`华为光猫配置文件加解密工具.exe`，输入文件选择`hw_ctree.xml`，输出文件设置为`hw_ctree.dec.xml.gz`，解密。
7.  解压`hw_ctree.dec.xml.gz`（OSX：`gzip -d hw_ctree.dec.xml.gz`）就能得到原始的配置文件了。
8.  编辑配置文件，将`UserLevel="1"`修改为`UserLevel=”0″`，即将`user`用户变更为管理员权限用户，可以看到你的登录密码也明文配置在了这里。

### 恢复华为出厂模式

9.  回到`telnet`
    ``` bash
    su
    shell
    restorehwmode.sh # 可以输入?查看所有可用命令
    exit
    reset
    ```
10. 等待路由器重启完成（如果不成功就拔电再插电试试）。这时候光猫恢复到了出厂模式，将电脑的静态IP改为`192.168.100.2`,网关`192.168.100.1`，浏览器访问`http://192.168.100.1`，就可以看到华为默认的登录界面了。
11. 用户名`telecomadmin`密码`admintelecom`登录，在系统工具栏，导出当前出厂配置留作备份，然后使用配置文件导入功能将刚才编辑过的xml配置文件（无需加密）导入即可。
12. 重新启动光猫后，再把IP改回`192.168.1.2`或者直接使用DHCP，访问`192.168.1.1`，使用`user`用户登录，之前的设置就都回来了，而且你作为超级管理员可以任意修改设置。

### 设置WAN桥接（交换机模式）

13. 进入WAN设置，将第二个 2_INTERNET_B_VID_3961的`WAN类型`改为`桥接WAN`。
14. 关闭WLAN等无用功能
15. 将光猫LAN1口接到自己的无线路由器WAN口上，在无线路由器中配置PPPoE拨号即可。

## 宽带测速

为了用足百兆宽带，参考[知乎导购网](https://www.zhihu.com/question/21739060/answer/60425087)在京东249入了一台[网件R6220](https://item.jd.com/1559009.html)在[SPEEDTEST](http://www.speedtest.net)测速，下行可以到`93.28Mbps`:

[![speedtest结果: 93.28Mbps](http://www.speedtest.net/result/6027628179.png "speedtest")](http://www.speedtest.net/my-result/6027628179)

华为`HG8346R`的LAN口都是百兆网卡，除去包头和包间距的损耗，峰值也就`94Mbps`，这个测速足够满意了。另外[联通使用单模光纤入户，理论最大速率155Mbps，换千兆猫意义也并不大](http://www.chinadsl.net/thread-123723-1-1.html)，所以就这么用着吧。

[mega]: https://mega.nz/#F!SMAEWb4B!4ZudCOxdlEzScMPmILPruw
[google-drive]: https://drive.google.com/open?id=0B0roQ8Jv0ddWWGhBczNFSEVaS1U
