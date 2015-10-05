---
layout: post
title: 碎碎念: ORM, python非阻塞编程, Go vs Elixir, MySQL on NUMA
date: 2015-10-05 23:20:00
categories: life
tags: orm python gevent golang elixir mysql numa
keywords: orm,non-blocking-programming
description: 算是一个年度总结吧

---

这篇文章也算是最近半年的一个总结，所以糅合了多个主题。鉴于可爱的歪果仁们已经有
了很棒的总结，我没打算把引用的链接再统统编译一遍，去读原文吧，我自己的吐槽反倒
无足轻重。

## 先说ORM

简言之，ORM sucks: [ORM - the Killer of Scalability][orm]。文中详述了ORM的七宗罪
（为刺激你点开链接我就不引用过来了嗯）。在我看来ORM是个反模式(anti-pattern)。

=======TL;DR; 碎碎念分割线，请无视下文=======

第一次接触ORM正是大学时代使用Java Web开发时。在很傻很天真，图样图森破的年纪，
努力去学习并迷恋起这种依靠复杂的配置、叠加各种设计模式以期"解耦"的编程方式。
然而随着开发的深入，我发现给SQL穿上这么一套华丽的Object衣服后，很多本来看似简
单的问题也会让我手足无措，这让我花在google上的时间远多于花在coding上的，而其中
有不少问题都是用plain SQL可以很简单就能做到的事情。这也难怪市面上大把大把的
JaveEE指南，Spring+Hibernate+BlahBlah权威教程之类了。这套衣服不光拖慢了开发的脚
步（懂SQL不够，还要精通ORM），甚至SQL走起路来也往往比裸奔慢得多。当时学识尚浅，
井中看到的天空是似乎是唯一真理，但我仍然隐约觉得，That's not the right way.

后面进入了PHP，使用hacking(用于分库分表路由)的ActiveRecord+PDO，比起Java世界
轻量了很多，在提供足够自由度的基础上尽量减轻了对开发的羁绊。再往后又来到了
Python web的世界(2015.1)，在这里sqlalchemy已然成为事实标准，而且它提供的ORM
看起来还算不赖，应付一个小项目足够。然而该来的总会来，java orm的历史在python
世界重演了。

sqlalchemy的世界分为两块大陆：core和ORM，贴个[最简明教程][sqlalchemy-tutorial]，
我准备将sqlalchemy ORM从我负责的Flask项目中移除，并基于sqlalchemy core封装更轻
量级的sql wrapper，从此Say farewell to all the ORM evils.

## Python non-blocking (web) programming

你猜我是不是又要写python sucks了？还真没错哈哈。在讲非阻塞前，先聊聊python的
并行/异步编程。

### Python并行编程

Python是一门"一个赛艇"的语言，它的`multiprocessing`和`threading`库也非常优雅，
然而，`GIL`(Global Interpretor Lock)却让`threading`处于一个相当尴尬的境地：如
果多线程还没有单线程效率高，那还用它作甚？相关介绍请参考
[Python中的GIL、多进程和多线程][python-gil]

在Web开发中，很多时候也会借助这两个库实现异步处理，不过在Flask开发框架下，这并
不总是件很容易的事情，这多是因为Flask的request context stack和application context
stack的原因：在派生新进程时，需要拷贝一份请求上下文到新进程中，具体方法就搜一下
StackOverFlow吧，之所以这样的原因可以参考[这篇文章][flask-request-stack]

### Gevent-Python中的协程库

这里我们说的是[协程](https://en.wikipedia.org/wiki/Coroutine)，不是`epoll`等非阻
塞的系统调用。关于`gevent`可以在[Gevent tutorial][gevent-tutorial]快速入门。生产
环境中的WSGI服务器，如`gunicorn`和`uWSGI`，都是可以通过`gevent`处理请求从而增大
单进程吞吐量的。

然而，`gevent`也并非看上去那么完美:
1. Monkey patching - an useful evil. 猴子补丁可以在不改变引用库的条件下使一些原
   生不支持非阻塞的库支持非阻塞的使用。可惜猴子补丁是error-prone的，尤其是引入多
   进程后，这个下文会讲。
2. 猴子补丁对C实现的库没有作用，而且python在当前线程执行C实现的库，这就会导致整
   个主线程阻塞在执行C代码上。比如使用MySQL-python执行MySQL查询。
3. 调试困难。`gevent`的上下文切换是通过`yield`完成的，在异常追踪和性能调优时，往
   往会力不从心。

在[How we use gevent to go fast][tune-gevent]中，介绍了针对`gevent`的调优策略。

### 并行+协程并发，看上去很美...

好吧，长话短说，`multiprocessing`在启用了monkey-patching的`gevent`环境中是不可用
的！有人对`multiprocessing`做了hack，推出了[gipc](https://gehrcke.de/gipc/)，在
一些简单的场景下，这么搭配也还不错。不过`gipc`还比较年轻，应用场景很有限。一些细
节文档中并没有做说明，这里列举两个：
1. 示例中都是主进程作为生产者，子进程为消费者。然而如果方向反过来以后，管道写端
   关闭会导致读端抛出`EOFError`。好在合理处理异常后不会造成管道数据错误。
2. 管道描述符对象(_GIPCHandle)一旦分配给某个进程，那么该对象在当前进程下就不可使
   用了，在处理进程同步时还是会有点小问题...

看来想要实现并行+协程并发，需要依赖很精巧的设计。可惜即便如此，我在实际开发中还
是遇到了各种诡异的问题，比如派生多进程后性能并没有多少提高（系统并未满载），甚至
还有性能下降，而原因却难以定位；比如脚本运行的好好地，可Ctrl-C中断执行时，中断处
理代码中的数据库逻辑(sqlalchemy session)却抱怨连接丢失等等等等。焦头烂额之下，我
决定放弃用python实现高性能处理脚本。

## Elixir vs Go

最近Elixir正吸引着越来越多的关注，甚至有人期待Elixir能再次带来当年Ruby on Rails
那样井喷式的革命。两门语言都是为了解决摩尔定律失效后，多核环境下的编程语言效率（
语言执行效率和开发效率）问题；通过语言级别的协程支持和进程调度，让编写并发程序和
编写一个函数一样简单且健壮。

关于两门语言的选择，应当视应用场景而定。一个简单（但不普适）的结论大概是Go更适合
命令行和底层应用的开发，无需虚拟机、直接编译为二进制单文件等特性在这方面具有天生
的便利；而Elixir更适合服务器应用开发：

>That being said, have you tried writing a web app in Go? You can do it, but it
isn't exactly entertaining. All those nice form-handling libraries you are used
to in Python and Ruby? Yeah, they aren't nearly as good. You can try writing
some validation functions for different form inputs, but you'll probably run
into limitations with the type system and find there are certain things you
cannot express in the same way you could with the languages you came from.
Database handling gets more verbose, models get very ugly with tags for JSON,
databases, and whatever else. It isn't an ideal situation. I'm ready to embrace
simplicity, but writing web apps is already pretty menial work, Go only
exacerbates that with so many simple tasks.

引用自 [The UNIX Philosophy and Elixir as an Alternative to Go][elixir-vs-go-on-web]


我的第一印象也是这样，Go更像C语言(类型的定义、命令式的语言风格等），而且Go本身也
将自己作为一门"System Language"来设计的。而Elixir更面向应用开发：继承Erlang/OTP
的衣钵(电信级的健壮性保证)，Ruby like的语法，管道符等诸多语法糖，代码的热更新部
署，逐步完善的包管理体制和开发工具链...这也是在我的技术路线图上Elixir要早于Go的
原因。

不光是WhatsApp, 还有一大票应用和游戏（比如使命召唤）选择了Elixir/Erlang, 推荐一
篇传教文：[Elixir - The next big language for the web][elixir-next-lang-for-web]

关于两个语言更多的讨论可以参考[Hacker News][elixir-vs-go-hacker-news]和
[Reddit][elixir-vs-go-reddit]

## MySQL在NUMA架构服务器上的SWAP问题

问题用一句话描述就是，MySQL server跑在一台
[NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access)架构的机器上，明
明还有不少空闲内存，但MySQL却开始了swap数据导致线上服务处于假死不响应的状态。
这里是[一份简要的中文介绍][numa-brief]，MySQL specific的话题建议将下面两篇文章读
完，Jeremy Cole大神解释的很细致：
1. [The MySQL swap insanity problem and the effects of the NUMA architecture][mysql-swap-insanity-and-the-numa-architecture]
2. [A brief update on NUMA and MySQL][mysql-numa-update]


[orm]: http://www.bigdatalittlegeek.com/blog/2014/3/18/orm-the-killer-of-scalability "ORM - the Killer of Scalability"
[sqlalchemy-tutorial]: http://solovyov.net/en/2011/basic-sqlalchemy/
[python-gil]: http://lesliezhu.github.io/public/2015-04-20-python-multi-process-thread.html "Python中的GIL、多进程和多线程"
[flask-request-stack]: http://www.zlovezl.cn/articles/charming-python-start-from-flask-request/ "Charming Python: 从Flask的request说起"
[gevent-tutorial]: http://xlambda.com/gevent-tutorial/ "Gevent指南"
[elixir-vs-go-on-web]: http://lebo.io/2015/06/22/the-unix-philosophy-and-elixir-as-an-alternative-to-go.html "The UNIX Philosophy and Elixir as an Alternative to Go"
[elixir-vs-go-hacker-news]: https://news.ycombinator.com/item?id=9761470
[elixir-vs-go-reddit]: https://www.reddit.com/r/elixir/comments/3c8yfz/how_does_go_compare_to_elixir/ "How does GO compare to Elixir?"
[elixir-next-lang-for-web]: http://www.creativedeletion.com/2015/04/19/elixir_next_language.html
[numa-brief]: http://cenalulu.github.io/linux/numa/ "NUMA架构的CPU -- 你真的用好了么？"
[mysql-swap-insanity-on-numa]: http://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/ "The MySQL swap insanity problem and the effects of the NUMA architecture"
[mysql-numa-update]: http://blog.jcole.us/2012/04/16/a-brief-update-on-numa-and-mysql/ "A brief update on NUMA and MySQL"
