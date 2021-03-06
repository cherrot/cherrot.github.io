---
layout: post
title: 设置git使用meld作为diff工具(支持目录比较)
date: 2012-09-18 23:02:49
categories: snippet
tags: git,meld,diff
description: 

---
搜了一下，抄来抄去的都不靠谱。抄的比较多的是这个方案：

> 因为meld只接受两个参数，而git diff会传递7个参数，因此需要编写个shell脚本转换一下： 
> #!/bin/sh
> meld $2 $5 
> 保存为git-meld，加可执行权限，然后设置git： 
> git config --global diff.external /PATH/TO/YOUR/git-meld 
> 这样，以后git diff的时候就可以直接用meld察看了。

然而这不符合我的需求。因为我不希望改变git diff的默认行为。实际上git提供了命令 git difftool 来完成这类工作，而且比引文中的方法更智能。

## 使用git difftool （git mergetool）

git difftool 和 git mergetool是专门提供给我们以用自己的工具进行diff和merge的命令。只要配置一下就可以使用了：

```
git config --global diff.tool meld #配置默认的difftool
git config --global merge.tool meld #配置默认的mergetool
```

然后输入命令 git difftool HEAD HEAD^1 看看？ :D

## 使用meld直接比较目录

虽然使用git difftool已经基本满足了我的需要，但还有个小问题：如果我要比较两次提交之间的差异时，difftool只能一个文件一个文件的比较，每次都要提示你是否打开这个文件，然后打开meld进行比较，当你关闭meld后，才会提示下一个差异文件。这样非常浪费效率。能不能直接利用meld的目录比较能力呢？ 搜了一下，果然有人把脚本写好了： [https://github.com/thenigan/git-diffall](https://github.com/thenigan/git-diffall) 下下来以后，进行如下配置：

```
git config --global diff.tool meld
git config --global alias.diffall /PATH/TO/YOUR/git-diffall
```

现在试试 `git diffall HEAD HEAD^1` ?

多谢**lxd**提出遇到的问题与我分享，上述配置会遇到错误 

```
Expansion of alias 'diffall' failed; '/xxxx/git-diffall/git-diffall' is not a git command
```

最方便的解决方法就是创建一个软链接在你的PATH路径之一里，比如：

```
ln -s /PATH/TO/YOUR/git-diffall ~/bin/git-diffall
```

然后配置

```
git config --global diff.tool meld #这一行必须配置，否则diffall不知道该使用哪个diff程序
git config –global alias.diffall git-diffall
```
