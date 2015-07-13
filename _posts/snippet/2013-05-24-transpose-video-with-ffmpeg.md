---
layout: post
title: ffmpeg大法好-视频旋转和高清转码示例
date: 2013-05-24 11:11:11
categories: snippet
tags: ffmpeg
description: 

---
手头有一个竖屏拍摄的视频（真诚建议不要这么做。。），导入到电脑上以后势必要把它旋转90°，可是没想到就这样简单的一个功能，尝试了N个非编软件（openshot, pitivi，还有坑爹的lives）后竟然没有一个可以满足我的要求。要么>是不支持自定义分辨率（openshot），要么是图像比例失调（pitivi），要么是奇慢无比（lives，感觉这货是面向工作站的大型非编工具，我等屌丝驾驭不了）。最后无奈，自己google，发现还是老外靠谱，一条命令拯救世界：
  
```bash
ffmpeg -i INPUT.AVI -vcodec libx264 -preset slower -crf 18 -threads 4 -vf transpose=2 -acodec copy OUTPUT.MKV
```

解释一下参数：
  
* `-i` 待转码文件
* `-vcodec` 选择视频编码。做过一番搜索，相比与MPEG2, MPEG4等，H.264是公认最好的高清编码格式，同时压缩率也高于MPEG4，所以我选择使用H.264(libx264)进行视频编码。
* `-preset` 选择编码预设，更慢=更好的视频质量，可选取值为 `ultrafast`,`superfast`, `veryfast`, `faster`, `fast`, `medium`, `slow`, `slower`, `veryslow`, `placebo`。`placebo`是没用的取值。
* `-crf` Constant Rate Factor，0~51之间取值，0为无损，23为默认取值，取值越大，视频整体质量越差。一般建议在18～28之间取值。18已经达到视觉无损的效果，即人眼几乎察觉不到和原片的差别。
* `-threads` 编码使用线程数，CPU几个核心就设置几个线程好了。  
* `-vf` 滤镜，我们只需要用到旋转滤镜`transpose=2`，`transpose`滤镜可取0-3，0为逆时针90°且垂直翻转，1为顺时针旋转90°，2为逆时针旋转90°，3为顺时针90°且垂直翻转。
* `-acodec` 音频编码，这里直接设置为`copy`保留原文件音频编码。
* 最后设置输出文件为OUTPUT.MKV
  
详细参数说明可以参考[FFmpeg and x264 Encoding Guide](http://ffmpeg.org/trac/ffmpeg/wiki/x264EncodingGuide)。
  
另外也找到一篇使用mencoder旋转视频的教程，可以参考这篇博文:[linux下快速旋转视频文件](http://blog.sina.com.cn/s/blog_71906a600100sp3v.html)。

