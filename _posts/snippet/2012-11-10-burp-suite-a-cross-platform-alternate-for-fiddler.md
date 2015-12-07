---
layout: post
title: Linux下Fiddler替代品：Burp Suite
date: 2012-11-10 23:02:49
categories: snippet
tags: fiddler,burp,http
description: 

---

刚开始接触前端开发时，师傅给我推荐了一枚神器 [Fiddler](http://www.fiddler2.com/)，最常用的功能莫过于自动替换HTTP响应为本地文件，从此写完js就可以方便地在本地调试而无需部署。只可惜此神器是用dotNet开发，仅能跑在Windows上面。 不过究其原理倒也不复杂，无非是作为一个HTTP代理服务器，根据匹配规则截取特定的请求和响应并按规则重新封包而已，我大Linux怎么可能没有相应的实现呢？于是信心满满的Google了一把，结果还真找到不少。一番试用下来，锁定了两个工具：

*   [Burp Suite](http://www.portswigger.net/burp/)
*   [WebScarab](https://www.owasp.org/index.php/Category:OWASP_WebScarab_Project)

其中后者是开源软件，前者是商业软件但提供免费版本。这两款软件都是用Java实现的，界面也很像。我个人使用过程中发现Burp更顺手一些，或者说，Burp是相对不那么难用的…… 本来想写一篇比较详细的教程的，后来发现有位仁兄做完了这份工作：[http://www.nxadmin.com/tools/689.html](http://www.nxadmin.com/tools/689.html) 那么我在这篇文章里只提供一个快速使用教程。 首先切换到Options选项卡设置软件的全局选项，比如你需要使用代理才能访问网络时，便需要在这里设置代理规则： [![](http://i.imgur.com/7J5Fp.png?1 "代理规则")](http://i.imgur.com/7J5Fp.png?1) 上图中我添加了4条代理规则，其中启用了3条。当访问第1和第3条的主机时将直接连接，其他主机使用第4条规定的代理连接网络。另外，Burp还支持Socks代理。 然后，前往Proxy选项卡，在options子标签中设置Burp本地代理。具体选项的含义请见上面的教程。一般保持默认就好。由于我需要替换服务器传回的某js脚本为我的本地版本，因此需要勾选intercept server responses的复选框"intercept if:"并把第一条"URL is in target scope"规则勾上，其次勾选update Content-Length，这样burp会自动根据更新HTTP报文的Content-Length字段。解释一下，"URL is in target scope"的意思是只拦截我们显式添加到监听域（下面会演示）的HTTP响应。 仍然在Proxy选项卡，确保intercept子标签中按下按钮"intercept is on"(若未按下会显示"intercept is off")。 接下来，打开浏览器，并且将浏览器的代理设置为Burp本地代理，代理类型是HTTP/HTTPS。 然后打开目标网页，这时，在Burp的target选项卡的site map子标签中，就列出了打开这个网页时所有的HTTP通信，在左侧找到你感兴趣的域或者在右侧找到具体的某个HTTP请求（如请求需要调试的js脚本），在右键菜单中点击add item to scope，这样就可以在scope子标签中看到这一条规则了。我们可以手动添加或编辑已有的scope条目，支持正则表达式。 现在刷新网页，Burp就会截取匹配scope规则的HTTP响应（我们刚刚设置了只截取HTTP响应），Proxy选项卡标红，我们便可以手动编辑服务器的HTTP响应啦。具体如下：

1.  在intercept子标签下，切换报文的查看模式为headers，这样便可以分别编辑HTTP报文头和报文体。
2.  清空报文的body，然后在报文body处右键，选择paste from file（从文件粘贴）。
3.  找到本地的js文件，确定。

最后点击forward按钮，burp就会将该HTTP报文发送给浏览器了。 可以看出，这款工具用起来比Fiddler的自动替换麻烦了很多…… 其实，burp的应用场景远不只是自定义HTTP响应（所以这个功能用起来是如此蛋疼……但总比没有好，对不），利用它可以做很多有趣的事情，比如暴力破解（hydra可能更适合你）、刷票、Web安全扫描等等，详情请参考上面贴的教程。
