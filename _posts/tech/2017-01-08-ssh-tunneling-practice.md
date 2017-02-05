---
layout: post
title: SSH隧道：内网穿透实战
date: 2017-01-08 19:00:00
categories: tech
tags: ssh linux
keywords: [linux, ssh, tunnel, proxy]
description: "介绍常见的ssh端口转发策略和应用场景"

---

## SSH支持的端口转发模式
正文开始前先用5分钟过一遍ssh支持的端口转发模式，具体使用场景在下一节详述。太长不看的可直接跳到下一节。

1.  ”动态“端口转发（SOCKS代理）：

    ``` bash
    ssh -D 1080 HostA  # D is for Dynamic
    ```
区别于下面要讲的其他端口转发模式，`-D`是建立在`TCP/IP`应用层的动态端口转发。这条命令相当于监听本地1080端口作为SOCKS5代理服务器，所有到该端口的请求都会被代理（转发）到HostA，就好像请求是从HostA发出一样。由于是标准代理协议，只要是支持SOCKS代理的程序，都能从中受益，访问原先本机无法访问而HostA可以访问的网络资源，不限协议（HTTP/SSH/FTP, TCP/UDP），不限端口。

2.  本地端口转发

    ``` bash
    ssh -L 2222:localhost:22 HostA  # L is for Local
    ```
这条命令的作用是，绑定本机2222端口，当有到2222端口的连接时，该连接会经由安全通道(secure channel)转发到HostA，由HostA建立一个到localhost（也就是HostA自己） 22端口的连接。  
如果上述命令执行成功，新开一个终端，执行`ssh -p 2222 localhost`，登录的其实是HostA。  
所以`-L`是一个建立在**传输层**的，**端口到端口**的转发模式，当然远程主机不仅限于localhost。  
后面的stdio转发和ProxyJump可以看做是本地端口转发的升级版和便利版（参见[OpenSSH netcat mode][ssh-stdio]）

3.  远程端口转发  

    ``` bash
    ssh -R 8080:localhost:80 HostA  # R is for Remote
    ```
顾名思义，远程转发就是在ssh连接成功后，绑定**目标主机**的指定端口，并转发到**本地网络**的某主机和端口：和本地转发相比，转发的方向正好反过来。  
假如在本机80端口有一个HTTP服务器，上述命令执行成功后，HostA的用户就可以通过请求`http://localhost:8080`来访问本机的HTTP服务了。

4.  stdio转发（netcat模式）与ProxyJump

    ``` bash
    ssh -W localhost:23 HostA
    ```
netcat模式可谓ssh的杀手特性：通过`-W`参数开启到目标网络某主机和端口的stdio转发，可以看做是组合了netcat(`nc`)和`ssh -L`。上述命令相当于将本机的标准输入输出连接到了HostA的telnet端口上，就像在HostA上执行`telnet localhost`一样，而且并不需要在本机运行telnet！  
既然是直接转发stdio，用来做ssh跳板再方便不过（可以看做不用执行两遍ssh命令就直接跳到了目标主机），所以在ProxyJump面世前（OpenSSH 7.3），`ssh -W`常被用于构建主机到主机的透明隧道代理，而ProxyJump其实就是基于stdio转发做的简化，专门用于链式的SSH跳板。

## 使用场景

### 建立代理

假设你在局域网A，HostB在局域网B，HostA有双网卡可以同时连接到局域网A和B。此时的你想要访问HostB上的web服务，便可以通过如下命令建立代理：

``` bash
ssh -D [::]:1080 HostA
```

这样，浏览器设置代理`socks5://localhost:1080`后，就可以直接访问`http://HostB`了。

当然，还可以通过这个代理ssh登录到HostB:

``` bash
ssh -oProxyCommand="nc -X 5 -x localhost:1080 %h %p" HostB
```
其中, `nc`需要BSD版（Ubuntu和OS X默认就是BSD版本），`-X 5`指定代理协议为SOCKS5，`-x`指定了代理地址，`%h %p`用于`ProxyCommand`中指代目标主机和端口。更多代理用法参见lainme姐的[通过代理连接SSH][ssh-over-proxy]和[通过代理使用GIT][git-over-proxy]。

`ssh -D`也是最基本的翻墙手段之一。

### 通过公网主机穿透两个内网

好，现在进入一种更复杂的情况：你和目标主机分属不同的内网，从外界都无法直接连通。不过好在这两个内网可以访问公网，你考虑通过一台公网机器建立两个内网之间的隧道。这种情况下，只能从内网建立到公网的连接，然而你需要的是从公网打通到内网：这不正是远程端口转发的用武之地嘛！

于是在目标网络，你吩咐现场人员帮你连通公网主机：

``` bash
# Host in LAN-Target
ssh -qTfNn -R 2222:localhost:22 HostA
```

`-qTfNn`用于告知ssh连接成功后就转到后台运行，具体含义见下一节解释。

现在，你只需要同样登录到跳板机HostA，就可以通过2222端口登录目标主机了：

``` bash
# in HostA, login target host
ssh -p 2222 localhost
```

#### 更进一步

两次ssh登录好麻烦，我们能否避免登录跳板机，从而直接穿透到目标主机呢？机智的你想到了组合`ssh -D`和`ssh -R`：

``` bash
# Host in LAN-Target
ssh -qTfNn -D :1080 localhost  &&  \
ssh -qTfNn -R [::]:12345:localhost:1080 HostA

# 也可以用下面的方式，虽然看起来更绕：
# ssh -R 2222:localhost:22 HostA   ssh -p 2222 -D 0.0.0.0:12345 localhost 
```

（因为要绑定公网的12345端口，请确保在公网主机的`/etc/ssh/sshd_config`里，配置了`GatewayPorts yes`，否则SSH Server只能绑定回环地址端口。）

上述命令在目标主机创建了SOCKS代理，并且映射到了公网HostA的12345端口，于是我们可以借助代理远程登录目标主机了：

``` bash
# Host in LAN-Client
ssh -oProxyCommand="nc -X 5 -x HostA:12345 %h %p" localhost
```

#### 限制端口

然而，直接在公网主机上暴露穿透到内网的SOCKS代理非常**不安全**。现在你想限制为仅暴露目标主机的ssh端口做穿透。这就可以通过ProxyJump(`ssh -J`)完成了。首先在目标主机上执行：

``` bash
# Host in LAN-Target
# 将公网HostA的2222端口转发到目标主机22端口，用于远程ssh登录
ssh -qTfNn -R 2222:localhost:22 HostA
```

成功后在本机执行：

``` bash
# Host in LAN-Client
# 通过ProxyJump跳板登录到目标主机，即使跳板机用户不能分配tty也没关系
ssh -J HostA -p 2222 localhost
```

如果OpenSSH版本&lt;7.3, 可以用stdio转发(`ssh -W`)代替，该命令会先登录HostA，继而转发本机stdio到HostA，所以接下来的ssh登录操作如同是在HostA完成一样：

``` bash
ssh -oProxyCommand="ssh -W %h:%p HostA" -p 2222 localhost
```

现在，对方的内网于你已是一览无余了 :D

### 通常意义的”跳板“

通常意义的”跳板“，指的是连接发起端A，经由跳板机B->C->D，连接到目标主机E的过程。连接和数据流都是单向的，比起上述情况反而简单了许多。这里不再赘述，只举两个简单的例子说明。更多示例参见[OpenSSH/Cookbook/Proxies and Jump Hosts][openssh-cookbook]

``` bash
ssh -L 1080:localhost:9999 HostA -t ssh -D 9999 HostB
```

这条命令会在登录HostA时，建立本机1080端口到HostA 9999端口的转发，同时在HostA上执行ssh登录HostB，同时监听9999端口动态转发到HostB。于是，所有到本机1080端口的连接，都被代理到了远程的HostB上去。

``` bash
ssh -J user1@Host1:22,user2@Host2:2222 user3@Host3
```

这条命令就是经由Host1, Host2，ssh登录到Host3的过程。


## Tips

### ssh执行为后台任务

`ssh -qTfNn`用于建立纯端口转发用途的ssh连接，参数具体含义如下：

- `-q`: quiet模式，忽视大部分的警告和诊断信息（比如端口转发时的各种连接错误）
- `-T`: 禁用tty分配(pseudo-terminal allocation)
- `-f`: 登录成功后即转为后台任务执行
- `-N`: 不执行远程命令（专门做端口转发）
- `-n`: 重定向stdin为`/dev/null`，用于配合`-f`后台任务

### 安全性

- 建议为端口转发建立专门的账户，使用随机密码（当然使用私钥登录更好），并且禁掉其执行命令的权限。最简单的方式为  

  ``` bash
  # add user tunnel-user for ssh port forwarding
  sudo useradd -m tunnel-user
  # generate 10 random passwords with 16 length
  pwgen -sy1 16 10
  # pick one password and set it to tunnel-user
  sudo passwd tunnel-user
  # disable shell for tunnel-user
  sudo chsh -s /bin/false tunnel-user
  ```

  更多可参考[Ask Ubuntu][ssh-user-only]
- 避免在公网直接暴露动态代理转发，**很危险**。 尽量远程端口转发到目标主机的ssh端口。这样需要远程接入的人可以自行ssh登录或打开本地Socks代理。

### 保持连接

客户端设置（`~/.ssh/config`）：

```
Host *
     ServerAliveInterval 180
```
每180秒向SSH Server发送心跳包，默认累积三次超时即认为失去连接。

服务器端同样可以设置心跳（`/etc/ssh/sshd_config`），作用同理：

```
ClientAliveInterval 180
```

### Windows 客户端

(我是个不喜欢贴图的人。。）以[PuTTY](http://www.putty.org/)为例，假如这台Windows主机在内网，我们要借助公网主机的远程端口转发建立隧道：

1.  和往常一样，在`Session`菜单输入公网主机的IP和SSH端口
2.  在`SSH`菜单里勾选`Don't start a shell or command at all`，以建立一个纯隧道连接（不需要登录shell）
3.  展开`SSH`菜单，进入`Tunnels`子菜单：
    1.  勾选`Remote ports do the same (SSH-2 only)`，使远程端口监听公网连接。
    2.  输入具体端口配置，比如`Source port`（也就是远程主机要监听的端口）填写22222，`Destination`填写`HostIP:22`，其中HostIP为内网中SSH服务器的IP。
    3.  选择`Remote`, `Auto`，表示建立远程端口转发。**点击`Add`添加配置**
4.  点击`Open`登录公网主机即可建立隧道。

[openssh-cookbook]: https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts "OpenSSH/Cookbook/Proxies and Jump Hosts"
[ssh-over-proxy]: https://www.lainme.com/doku.php/blog/2011/01/%E9%80%8F%E8%BF%87%E4%BB%A3%E7%90%86%E8%BF%9E%E6%8E%A5ssh "透过代理连接ssh"
[git-over-proxy]: https://www.lainme.com/doku.php/blog/2011/07/%E9%80%9A%E8%BF%87%E4%BB%A3%E7%90%86%E4%BD%BF%E7%94%A8git "透过代理连接git"
[ssh-user-only]: http://askubuntu.com/questions/48129/how-to-create-a-restricted-ssh-user-for-port-forwarding "Howto create SSH user only for tunneling"
[ssh-stdio]: https://blog.rootshell.be/2010/03/08/openssh-new-feature-netcat-mode/ "OpenSSH netcat mode"
