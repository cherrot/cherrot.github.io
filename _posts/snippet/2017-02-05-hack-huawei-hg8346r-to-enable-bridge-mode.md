---
layout: post
title: 破解华为HG8346R/HG8347R光猫 启用有线桥接
date: 2017-02-05 18:58:00
categories: snippet
tags: 路由器
description: 华为`HG8346R`光纤路由器自带的功能限制太多，而且无线路由能力很弱，破解后可以开启有线桥接模式（交换模式），使用单配的无线路由器拨号连接，最大化发挥网络性能。本文适用于OSX, Windows 和 Linux系统。

---

## 修订记录

- 2018.02.23: 补充了破解操作细节，并验证了`HG8347R`的破解。

本文兼顾多款光猫型号，并兼容Windows、OSX/Linux，内容略杂，请按目录跳跃浏览。

## 为何破解

1.  联通定制版无线限制了无线最多5个客户端，根本无法满足日常使用。
2.  无线实时性差，不支持5GHz频段，带宽也不高，虽然理论能到300Mbps，实际差得远。
3.  定制版限制太多，很多设置作为普通用户无法修改，甚至连无线SSID都强制以`CU_`开头，日了狗。

由于定制版无法开启WAN口桥接，所以需要恢复光猫原生系统并使用管理员账户设置。（WAN口桥接模式，其实就是将光猫作为交换机使用，转发二层数据帧比路由三层IP包快的多，减少性能损耗。）


## 准备工作

本文适用于（北京）联通定制版的华为`HG8346R`、`HG8347R`光纤接入终端。软件版本（可以登录web界面在“状态”中找到）：`V3R015C10S109`, `V3R016C10S135`, `V3R017C10S103`，后文使用R15, R16, R17指代。

需要下载华为ONT维护工具包用于开启维护模式和加解密配置文件，并开启tftp服务器用于备份路由器配置。可以在我的[Google Drive][google-drive]或[Mega网盘][mega]下载。其中ONT工具shasum: `beb12c70bf2871de4254176be42b6748176a6471`。

如果软件版本为R17，需要额外下载固件包：[Google Drive][rom-gdrive]或[Mega][rom-mega]。shasum: `7029508ec8f7e6a78660892557b8acdb41317803`。


### Windows

工具包里带了`tftpd32`和`tftpd64`程序，直接运行就可以启动tftp服务器。

另外建议下载`putty.exe`运行`telnet`命令，比如`telnet 192.168.1.1`，那么Host Name (or IP address)填写`192.168.1.1`，Connection Type选择Telnet，点击"Open"即可。

### OSX / Linux

华为的维护工具包是Windows程序，需要安装`wine`模拟器：

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

OSX用户**不要使用系统自带的tftpd服务，基于tftpd的GUI版软件也不行（比如tftpserver）**，因为要想上传文件（`put`），必须更改tftpd的启动参数（`-s`），但`OS X EI Capitan`禁用了编辑服务配置文件的权限，除非重启进入Recovery模式。另外我尝试直接运行`tftpd`也没戏，折腾个把小时还不如3行Python代码来的简单。

## Let's hack!

1.  拔出光纤，电脑用网线连到光猫**LAN1**口，我们假定电脑获取的IP是`192.168.1.2`
2.  浏览器进入`192.168.1.1`，按照光猫背面的用户名密码登录，在网络中关闭无线功能。
3.  重启光猫（光猫后部有按钮开关），启动完成后，光猫的电源灯常亮，LOS灯红色闪烁（表示未接入光纤），LAN1灯闪烁（表示有数据流量）。

### R17版本：降级固件进入维护模式

R17版本必须降级固件到R16才能成功进入维护模式，直接尝试开启维护模式会发现所有灯闪烁几遍后全灭，光猫无响应（重启恢复）。

1.  运行`华为ONT组播版本配置工具.exe`（OSX执行`wine 华为ONT组播版本配置工具.exe`，下文不再赘述），选择“升级”，选择计算机本地网卡`192.168.1.2`，点击“浏览”选择刚刚下载的`hg8347r16-rom.bin`，其他参数不变。
2.  点击“启动”，这时光猫上所有灯会一起闪烁，刷入过程大概需要8分半。等到所有灯常亮，点击停止（工具下方的进度条没有任何意义，不用管它）。
3.  重启光猫，完成后便已开启`telnet`服务。
4.  浏览器登录<http://192.168.1.1>，在安全设置中禁用防火墙，以防`telnet`无法连接。

### R15, R16版本：开启维护模式

1.  运行`华为ONT组播版本配置工具.exe`，选择本地网卡`192.168.1.2`，其他保持默认，点击`启动`按钮。
2.  耐心等待2分钟左右，如果所有灯常亮，说明成功开启了维护模式。点击`停止`并关闭工具。
3.  重启光猫，完成后便已开启`telnet`服务。

### 备份配置

1.  打开终端或命令行：

    ``` bash
    telnet 192.168.1.1

    root #（Login用户名）
    admin #（密码不回显）
    ```
2.  确保本机tftp服务已开启，在`telnet`中执行：

    ``` bash
    backup cfg by tftp svrip 192.168.1.2 remotefile hw_ctree.xml
    ```
    如果命令执行成功，你设定的tftp目录下（tftp-dir）便有了`hw_ctree.xml`配置文件了。
3.  运行`华为光猫配置文件加解密工具.exe`，输入文件选择`hw_ctree.xml`，输出文件设置为`hw_ctree.dec.xml.gz`，解密。
4.  解压`hw_ctree.dec.xml.gz`（OSX：`gzip -d hw_ctree.dec.xml.gz`）得到`hw_ctree.dec.xml`。

### 恢复华为出厂模式

1.  回到`telnet`

    ``` bash
    su
    shell
    restorehwmode.sh # 可以输入?查看所有可用命令
    exit
    reset
    ```
2. 等待路由器重启完成（如果不成功就直接按开关重启）。这时候光猫恢复到了出厂模式，将电脑设置为静态IP`192.168.100.2`，子网掩码`255.255.255.0`，网关`192.168.100.1`，浏览器访问<http://192.168.100.1>，就可以看到华为默认的登录界面了。
3. 用户名`telecomadmin`密码`admintelecom`登录，在系统工具栏，导出当前配置留作备份。

### 导入配置

简单粗暴式：备份并编辑`hw_ctree.dec.xml`，搜索`X_HW_WebUserInfoInstance`，修改为`<X_HW_WebUserInfoInstance InstanceID="1" UserName="用户名" Password="密码" UserLevel="0" Enable="1" ModifyPasswordFlag="0" PassMode="0"/>`即可设置管理员账户。

小心翼翼式：备份并编辑刚刚导出的`hw_ctree.xml`，把`hw_ctree.dec.xml`中的`<WANDevice NumberOfInstances="1">...</WANDevice>`, `<LANDevice NumberOfInstances="1">...</LANDevice>`, `<VoiceService NumberOfInstances="1">...</VoiceService>`, `<X_HW_IPTV....../>`的节点内容完整替换过来。并修改`X_HW_WebUserInfoInstance`中的用户属性（`UserLevel="0"`表示管理员用户，`PassMode="0"`表示明文密码）

将修改后的配置在浏览器中重新导入，等待光猫重启即可生效。将本机网络重新设为DHCP，待光猫重启完成后即可登录<http://192.168.1.1>进行后续设置。

### 设置WAN桥接（交换机模式）

1. 进入WAN设置，将第二个 2_INTERNET_B_VID_3961的`WAN类型`改为`桥接WAN`。
2. 关闭WLAN，防火墙等无用功能
3. 将光猫LAN1口接到自己的无线路由器WAN口上，在无线路由器中配置PPPoE拨号即可。

### IPTV

如果要使用联通的IPTV机顶盒，可以重新打开无线WLAN1，并将WAN设置中的IPTV连接端口绑定到SSID1，这样机顶盒就可以使用光猫的无线信号观看IPTV了。

## 宽带测速

### HG8346R

参考[知乎导购网](https://www.zhihu.com/question/21739060/answer/60425087)在京东249入了一台[网件R6220](https://item.jd.com/1559009.html)在[SPEEDTEST](http://www.speedtest.net)测速，下行可以到`93.28Mbps`:

![speedtest结果: 93.28Mbps](http://www.speedtest.net/result/6027628179.png "speedtest")

华为`HG8346R`的LAN口都是百兆网卡，除去包头和包间距的损耗，峰值也就`94Mbps`，这个测速足够满意了。~~另外[联通使用单模光纤入户，理论最大速率155Mbps，换千兆猫意义也并不大](http://www.chinadsl.net/thread-123723-1-1.html)，所以就这么用着吧。~~

### HG8347R

联通去年给自动升级到了200M带宽，升级光猫后LAN1口可到千兆，下行能到`186.45Mbps`:
![200M测速: 186.45Mbps](http://www.speedtest.net/result/7083094506.png "speedtest")

美滋滋~


## 致谢

本文参考了以下教程，在此感谢作者的无私分享：

- [北京联通华为光猫HG8346R破解改桥接-什么值得买](http://post.smzdm.com/p/441624/)
- [北京联通华为HG8347R破解-ChinaADSL](http://www.chinadsl.net/forum.php?mod=viewthread&tid=128938)

[mega]: https://mega.nz/#F!SMAEWb4B!4ZudCOxdlEzScMPmILPruw
[google-drive]: https://drive.google.com/open?id=0B0roQ8Jv0ddWWGhBczNFSEVaS1U
[rom-mega]: https://mega.nz/#!uM42hITa!uw9BkxOLxPKerZrNMkZ9LanIlUrE8w9eZxZbHFc292g
[rom-gdrive]: https://drive.google.com/open?id=1tOo7CLWCtk0-T-oonVFB-fEUtnkiIB5Z
