---
layout: post
title: "在git hook中执行后台进程"
date: 2015-12-07 21:39:00
categories: snippet
tags: git
keywords: 
description: "howto: run background (time consuming) command in git hooks like post-receive"

---

18:46 < tg2arch> [Isaac Ge] 我在写根据当前 Git 目录下的 .gitignore 内容来查找出被引入 Git 的垃圾文件
18:46 < tg2arch> [Isaac Ge] 的脚本
18:47 < tg2arch> [farseerfc] 爲什麼不直接
18:47 < tg2arch> [farseerfc] git clean
18:48 < tg2arch> [Isaac Ge] 都被引入了
18:48 < tg2arch> [Isaac Ge] 都被 tracked 了
18:49 < tg2arch> [farseerfc] git rm --cached -n
18:50 < tg2arch> [farseerfc] git ls-files -ci --exclude-standard -z | xargs -0 git rm --cached 
