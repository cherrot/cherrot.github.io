---
layout: post
title: UNIX下多进程共享监听socket的方式
date: 2016-01-05 01:08:00
categories: tech
tags: [linux, UNIX, socket, nginx]
keywords: linux,fork,socket,UNIX,nginx
description: how do multiple processes share one socket

---

## 黑科技：Node.js的cluster不科学啊

上周同事问我一个问题，为什么使用`Node.js`的`cluster`时可以在worker进程中随意用
`server.listen(1234)`来监听某个端口而不冲突？这在直观看来相当不科学啊，不然就
不会总是有UNIX网络编程新手问为什么bind socket时遇到"Address in use"的问题了。

今天小搜一下，果然在[StackOverFlow][how-cluster-works-1]有知音：

>The worker processes are spawned using the [child_process.fork][] method, so
that they can communicate with the parent via IPC and pass server handles back
and forth.

>When you call server.listen(...) in a worker, it serializes the arguments and
passes the request to the master process. If the master process already has a
listening server matching the worker's requirements, then it passes the handle
to the worker. If it does not already have a listening server matching that
requirement, then it will create one, and pass the handle to the worker.

上述答案援引自[Cluster官网][how-cluster-works-2]，不过第二段在官网中已经删掉，
备忘在这里以更清晰的理解其中原理。所以看上去是违反了UNIX常识，实际上只是对
`listen`方法做了包装，最终还是由master进程监听端口，并派发/路由请求到对应的
worker进程（默认使用round-robin方式路由请求）

## UNIX科普：file descriptor可以共享和传递

然而我的好奇心并未止于此。多年没碰UNIX网络编程的我依稀记得一个file descriptor
只能由一个进程持有，可是为什么这样规定？像`nginx`,`node.js`这种一个master派发
若干个worker进程后，master中创建的file descriptor (TCP socket)在worker进程中
不能被使用吗？Google一下才知道，原来我的直观印象是错的，file descriptor当然可以
[共享和传递][share-or-pass-fd-among-procs]：

>Yes you can, using sendmsg() with SCM_RIGHTS from one process to another:

>>SCM_RIGHTS - Send or receive a set of open file descriptors from another
process. The data portion contains an integer array of the file descriptors.
The passed file descriptors behave as though they have been created with
dup(2).
http://linux.die.net/man/7/unix

>that is not the typical usage tho. more common is when a process inherits
sockets from its parent (after a fork()). any file handles (including sockets)
not closed will be available to the child process. so the child process
inherits the parent's sockets.

也就是说,`fork`后的子进程天生就能继承父进程创建的file descriptor。这也是当下
`nginx`等web server处理和派发请求的方式。

顺便贴个链接复习一下UNIX网络编程的基础(en)：
[单连接, fork, select: 三种监听socket的方式][socket-tutorial]

## 更进一步：通过SO_REUSEPORT享受Linux内核提供的进程间负载均衡！

好奇心驱使我仍未止步，在搜索`cluster`工作原理时，无意中发现BSD后来推出了一个黑
科技一般的Socket Option: `SO_REUSEPORT`，有了这个选项，我们可以让任意进程（
linux下限制必须是同一用户的进程）同时`bind`相同的source address和port而不报错！
切记不要用成`SO_REUSEADDR`，这两个Option目的不同。Linux在3.9版本以后正式支持了
`SO_REUSEPORT`，并且提供了**进程间负载均衡**的隐形福利：

>Additionally the kernel performs some "special magic" for SO_REUSEPORT sockets
that isn't found in any other operating system so far: For UDP sockets, it
tries to distribute datagrams evenly, for TCP listening sockets, it tries to
distribute incoming connect requests (those accepted by calling accept())
evenly across all the sockets that share the same address and port combination.
That means while it is more or less random which socket receives a datagram or
connect request in other operating systems that allow full address reuse, Linux
tries to optimize distribution so that, for example, multiple instances of a
simple server process can easily use SO_REUSEPORT sockets to achieve a kind of
simple load balancing and that absolutely for free as the kernel is doing "all
the hard work" for them.

更多细节请深度阅读[StackOverFlow上的答案][SO_REUSEPORT]，
虽然略长，但很浅显易懂，强烈建议像我这样的非专业底层人士通读一遍。

所以直觉上来讲，使用`SO_REUSEPORT`，让内核处理负载均衡应该比让master进程负责
监听和派发请求到对应worker进程的方式更有效率，果不其然，`nginx`在1.9.1版本中的
Socket Sharding就是通过`SO_REUSEPORT`实现的:

![before](https://assets.wp.nginx.com/wp-content/uploads/2015/05/Slack-for-iOS-Upload-1-e1432652484191.png)

(默认策略：master监听socket，workers使用`accept_mutex`竞争request connections)

![after](https://assets.wp.nginx.com/wp-content/uploads/2015/05/Slack-for-iOS-Upload-e1432652376641.png)

(启用Socket Sharding后)

benchmark能达到默认策略的3倍性能(req/s和latency):

![benchmark](https://assets.wp.nginx.com/wp-content/uploads/2015/05/reuseport-benchmark.png)

详见[官方Blog][socket-sharding-in-nginx]。顺便也科普一下[nginx的架构和原理][inside-nginx]

所以，未来版本的`cluster`或许也可以扔掉依靠IPC通信的黑魔法，直接使用
`SO_REUSEPORT`，以获得更劲爆的性能。

[how-cluster-works-1]: http://stackoverflow.com/questions/9830741/how-does-the-cluster-module-work-in-node-js "How does the cluster module work in Node.js?"
[how-cluster-works-2]: https://nodejs.org/api/cluster.html#cluster_how_it_works "How Cluster Works"
[share-or-pass-fd-among-procs]: http://stackoverflow.com/questions/1997622/can-i-open-a-socket-and-pass-it-to-another-process-in-linux "Can I open a socket and pass it to another process in Linux"
[socket-tutorial]: http://www.jasonernst.com/2011/03/22/tutorial-sockets-3-ways-to-listen/ "Tutorial: sockets – 3 ways to listen"
[SO_REUSEPORT]: http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t "Socket options SO_REUSEADDR and SO_REUSEPORT, how do they differ? Do they mean the same across all major operating systems?"
[socket-sharding-in-nginx]: https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/ "Socket Sharding in NGINX Release 1.9.1"
[inside-nginx]: https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/ "Inside NGINX: How We Designed for Performance & Scale"
