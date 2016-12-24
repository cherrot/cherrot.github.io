---
layout: post
title: Cherrot's Daily
description: Happy Hacking ;-)

---

## whoami

Software engineer at Megvii

Happy hacking ;)

## 存在感

* 远离社交网络
* [GitHub](https://github.com/cherrot)
* [图虫](http://cherrot.tuchong.com/)：从D90到D7000到D610，从18-200到50和35：我似乎体会到写了多年代码却依旧没有入门是怎样一种体验了。

### 2016.04.07 LVM

[LVM](https://wiki.archlinux.org/index.php/LVM)是个好东西, 动态扩容磁盘不费劲儿:

1. `vgs/vgscan/vgdisplay`, `pvs/pvscan/pvdisplay`,`lvs/lvscan/lvdisplay`, `lvmdiskscan`查看.
2. `vgcreate`(创建 volume group.
3. `pvcreate /dev/sdX`创建 physical volume.
4. `vgextend vg0 /dev/sdX`扩展volume group.
5. `lvresize -l +100%FREE --resizefs vg0/lv0`

### 2016.03.01 离线安装pip依赖

```
# 在线环境中下载依赖:
pip install -r requirements.txt -d /tmp/packages
# 使用本地目录安装依赖:
pip install -r requirements.txt --no-index --find-links file:///tmp/packages
```


### 2016.01.04 awk按行去重

```
awk 'NR==FNR{m[$0]}NR>FNR{if(!($1 in m)) print $0}' b.txt a.txt
```

### 2015.11.25 /tmp overflow

see [fatal error: error writing to /tmp/ccwAjc9Z.s: No space left on device](http://sharadchhetri.com/2014/02/25/fatal-error-error-writing-tmpccwajc9z-s-space-left-device/)

```
df -h
...
overflow        1.0M     0  1.0M   0% /tmp
```

```
umount /tmp
mount -a
```

### 2015.11.24 CORS 对30x并不友好

线上通过将`video.test.com/somevideo?frame=123` 301 redirect 到
`image.test.com/somevideo_123`来模拟了一个视频服务器用来做一些好玩的事情。由于
某些图片服务可能用到`canvas`，就给全域的`<img>`标签统一加了
`crossorigin=anonymous`属性（不然当canvas请求图片命中浏览器缓存且没有CORS头时就
会被跨域限制block掉）。然而我在请求这个"video"时发现redirect后的请求被block了，
仔细一看发现原来是Chrome设置了redirct后的请求头`Origin`为`null`...
你也觉得很坑爹是不是？然而google一下发现这还不能说是
[bug](https://code.google.com/p/chromium/issues/detail?id=154967)，因为人家
[W3C就这么规定](http://www.w3.org/TR/cors/#redirect-steps)的(见第6步)，我感到下
体一阵刺痛。。

如果对安全性不那么敏感，可以在`image.test.com`上将`null`设置为合法的
"Access-Control-Allow-Origin", 这样，任何redirect到这个域名的请求都被认为是跨域
安全的了。否则就只能退而求其次，将不需要canvas的`<img>`标签统一去掉跨域属性
(`crossorigin`)，仅在canvas场景下给图片URL加一个自定义参数避免命中浏览器缓存。

### 2015.11.18 redis内存分析 + csv终端可视化

redis内存分析工具[redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools):

```
pip install rdbtools
rdb -c memory /var/lib/redis/dump.rdb > rdb.csv
column -s, -t < rdb.csv|less -#2 -N -S
```

总之`column`是个神奇的工具

### 2015.11.02

`libav` [抽取关键帧](https://libav.org/documentation/avconv.html#Video-and-Audio-grabbing): 

```
avconv -ss 10 -t 30 -i video.mp4 -r 24.98 -f image2 /tmp/matrix.mp4_%d

-ss: 取样起始时间
-t: duration
-r: 取样率（不大于原始FPS的值）
-f: 强制format
```

### 2015.08.10
利用`shuf`和`wc`按比例随机抽样文件行，简单高效：

```
file=Your_File
shuf -n $((`wc -l < $file`/10)) $file
```

### 2015.08.09
买了块PCI-E接口的无线网卡，linux下直接可用。并且意外的找到了折磨了我一年的Chrome地址栏卡顿的[问题原因](http://suselinks.us/how-to-fix-slow-typing-in-chrome-addressbar-in-linux/)。真是哔了狗了……当时是`darktable`的某些UI模块中文显示方块，所以改动了fontconfig，将文泉驿放到了`sans-serif`的首选位置，没想到这导致了Chrome UI因无法命中字体缓存而频繁调用`fontconfig`。唉，深深感觉被愚弄了，看来还是得学点调试技巧 ;)

### 2015.08.03
今天的晚霞好美，云层时而金黄时而灰黑。下班时夕阳西下，唯有无边无际的黑云压城，从街边路灯下望去，特别有史诗感。

今天前端遇到访问阿里云存储的跨域问题，设置CORS后发现不管用啊~提工单跟人家BB了半天才知道阿里云OSS根据请求中是否有`Origin` header来决定是否返回CORS headers，这的确符合规范，可我是`<img>`标签，不是ajax肿么办。。。最后在[MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_enabled_image)上发现原来只需要在`<img>`标签上加一个`crossorigin`即可解决，害我苦思冥想了各种猥琐方案浪费了一下午，原来订协议的早就想到并且提供了这么优雅的解答。这时抬头看到窗外的晚霞，身心舒畅。

最近尝试使用`fluentd`+`elasticsearch`部署一个log存储检索系统。目前尚在踩坑阶段，不过`fluentd`的社区很活跃，提交的Pull Request竟然有望被合并，撒花~ 
相比而言`elasticsearch`就更偏封闭，使用它更多是因为找不到更合适的而已吧。等坑基本踩完后再写篇总结好了。

### 2015.07.12
今天找了一天房子，然并卵，对不喜欢的事情我还真是重度拖延症。

《天使面庞 The Face of an Angel》是部佳作。

>Is life always this hard, or is it just when you're a kid? 

>Always like this.
