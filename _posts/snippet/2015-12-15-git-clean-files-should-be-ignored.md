---
layout: post
title: "Git清理本应ignore却被track的文件"
date: 2015-12-15 19:07:00
categories: snippet
tags: git
keywords: 
description: ""

---

来自 #archlinux-cn @freenode IRC 2015-12-15

```
18:46 < tg2arch> [Isaac Ge] 我在写根据当前 Git 目录下的 .gitignore 内容来查找出被引入 Git 的垃圾文件
18:46 < tg2arch> [Isaac Ge] 的脚本
18:47 < tg2arch> [farseerfc] 爲什麼不直接
18:47 < tg2arch> [farseerfc] git clean
18:48 < tg2arch> [Isaac Ge] 都被引入了
18:48 < tg2arch> [Isaac Ge] 都被 tracked 了
18:49 < tg2arch> [farseerfc] git rm --cached -n
18:50 < tg2arch> [farseerfc] git ls-files -ci --exclude-standard -z | xargs -0 git rm --cached 
```
