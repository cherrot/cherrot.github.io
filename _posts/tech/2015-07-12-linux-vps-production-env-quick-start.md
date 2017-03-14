---
layout: post
title: Linux Web生产环境速配指南
date: 2015-07-12 23:02:49
categories: tech
tags: linux git ubuntu
keywords: linux,git
description: Linux生产环境速配指南，使用git作为自动部署工具

---
## Linux账户和SSH配置

1.  使用`adduser`脚本（Ubuntu系统）添加用户master并设置明码：

    ```bash
    adduser master
    passwd master
    ```

    `adduser` 会交互式提示用户输入必要的信息，创建用户和与用户同名的组，并创建home目录。如果需要更多自定义选项，可以使用原始命令`useradd`:

    ```bash
    # OPTIONAL, create a user group.
    groupadd mygroup
    useradd -m -g mygroup -s /bin/zsh master
    passwd master
    ```

    这里创建了用户`master`，默认组为`mygroup`(此参数可省略)，默认shell为`zsh`。注意，出于[安全考虑](https://wiki.archlinux.org/index.php/Users_and_groups#Example_adding_a_user)并不建议修改用户默认组，但这确实是一个比较简便的账户共享方式。

2.  添加`sudoer`权限：

    ```bash
    adduser master sudo
    # on other Linux, you may need run visudo and uncomment the `%wheel ALL=(ALL) ALL` line first.
    # see: https://wiki.archlinux.org/index.php/Sudo_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
    # gpasswd -a master wheel
    ```

3.  在本地添加ssh配置`~/.ssh/config`:

    ```nginx
    Host *
        ServerAliveCountMax 20
        ServerAliveInterval 240
    Host YourHostName
        HostName XXX.XXX.XXX.XXX
        Port 22
        User master
    ```

4.  添加ssh公钥认证

    ```bash
    ssh-keygen -t rsa -C "email@example.com" #创建ssh密钥对，已有ssh密钥可忽略
    #公钥默认保存于 ~/.ssh/id_rsa.pub
    ssh-copy-id -i ~/.ssh/id_rsa.pub cherrot #添加公钥到服务器
    ```

    添加成功后重新连接，检查是否已经使用公钥认证（只要不再提示输入密码就可以登陆，那就表示公钥验证成功）。登录成功后，我们退出root登陆的session，下面的操作都基于master用户session。
    这里我遇到一个小插曲，由于我更新了个人的ssh密钥对，导致ssh认证失败：`Agent admitted failure to sign using the key`。重新注册一下自己的私钥即可解决：

    ```bash
    ssh-add ~/.ssh/id_rsa 
    ```

5.  修改服务器sshd配置 `/etc/ssh/sshd_config`：
在修改服务器sshd配置前，需要确保使用个人密钥可以成功登录（不然当我们禁用密码登录，密钥登录又失败时，会死得很惨的……）
修改以下字段禁用root登录及密码验证:

    ```
    PermitRootLogin no
    PasswordAuthentication no
    ```

    保存，重启sshd服务（Ubuntu直到14.10才支持systemd，遗憾）：

    ```bash
    sudo service ssh restart
    ```

## 生产环境配置(nginx+git)
本节参考[凤凰君的 Ubuntu 服务器配置简易指南][phoenix]。

注意，如果线上部署环境需要多人共享，那么建议单独为git部署创建一个用户`git`，以避免各种奇奇怪怪的权限问题:D

### 服务器端git环境
1.  首先创建裸仓库：

    ```bash
    mkdir -p ~/repo/YourRepo.git && cd ~/repo/YourRepo.git
    git init --bare --shared=group 
    ```

    如果仓库已存在，可用`git config --bool core.bare true`完成到bare仓库的转换

2.  添加`post-receive`钩子，用于push代码后自动部署文件。创建并编辑`hooks/post-receive`:

    ```bash
    #!/bin/sh
    # Change `master` to your release branch
    GIT_WORK_TREE=$HOME/www/YourSite git checkout -f master
    cd $HOME/www/YourSite/
    # find . -type f -exec chmod 644 {} \;
    # find . -type d -exec chmod 755 {} \;
    # Maybe you have some config files which were not in your repo
    ln -sf $HOME/data/YourSite/* ./
    exit
    ```

3.  创建必要目录并给`post-receive`添加可执行权限。

    ```bash
    mkdir -p ~/www/YourSite
    mkdir -p ~/data/YourSite
    chmod +x hooks/post-receive
    ```

### 本地git环境:
只需要添加remote仓库地址即可：

```bash
git remote add deploy master@YourHostName:repo/YourRepo.git
git push -u deploy master
```

现在看看是不是已经成功部署到指定目录了？

### 服务器nginx:
编辑文件 `/etc/nginx/nginx.conf`

```nginx
# 修改 NGINX 进程数量，根据 CPU 核心数而定
worker_processes 4;
# events 部分，修改 NGINX 进程连接数，同时打开多线程(默认被注释，取消注释即可)
worker_connections 768
multi_accept on;

    # http 部分，关闭服务器版本号(默认被注释，取消注释即可)
    server_tokens off;
    # 打开 GZIP 减少数据量，会稍微增加 CPU 负担。这些语句直接取消注释即可。
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
    #更简洁清晰的日志格式
    log_format clean '$status\t$request_time\t$remote_addr'
	'\t\t$time_local\t$body_bytes_sent'
	'\t"$request"\t"$http_referer"';

    # default log format:
    #log_format combined '$remote_addr - $remote_user [$time_local] '
    #    '"$request" $status $body_bytes_sent '
    #    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log clean;
    error_log /var/log/nginx/error.log;
```

下文列出了常用Web站点的nginx配置示例，Ubuntu系统下默认将所有虚拟主机配置放置在`/etc/nginx/sites-available/`目录，而把生效配置放置在`/etc/nginx/sites-enabled/`下。更新配置后需要平滑重启nginx和web服务（如`php5-fpm`）以生效。

注意，`listen`字段存在平台差异性，如果`nginx`抱怨bind失败，请仔细阅读[nginx官方文档][nginx]

```bash
ln -sf /etc/nginx/sites-available/* /etc/nginx/sites-enabled/
sudo service php5-fpm restart
sudo service nginx restart
# Maybe better: 
# sudo nginx -s reload
```

#### nginx for PHP示例:

```nginx
server {
    # 同时侦听 IPv4+IPv6 80端口：
    listen [::]:80 ipv6only=off;

    server_name cherrot.com www.cherrot.com;

    # access_log   /var/log/nginx/cherrot.com.access.log clean;
    # error_log    /var/log/nginx/cherrot.com.error.log;

    root /home/master/www/YourSite;
    index index.php index.html;
    location / {
        try_files $uri $uri/ /index.php?$args; 
    }
    
    error_page 404 /404.html;
    # redirect server error pages to the static page /50x.html
    #
    # error_page 500 502 503 504 /50x.html;
    # location = /50x.html {
    # }

    location ~ \.php$ {
        # fastcgi_split_path_info ^(.+\.php)(/.+)$;
        # NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
        try_files $uri =404;
        # fastcgi_index index.php;
        include fastcgi_params;

        fastcgi_pass unix:/var/run/php5-fpm.sock;
    }
}
```

#### nginx for python示例（以flask on uwsgi为例）:

```nginx
server {
    listen [::]:80 ipv6only=off;
    server_name www.yoursite.com;
    root /home/master/www/another;
        
    location / {
        # The path of your uwsgi unix socket
        uwsgi_pass unix:/home/master/uwsgi.sock;
        include uwsgi_params;
    }
        
    location /static/ {
        # Only absolute path works.
        # alias /static/;  
        # alias /home/master/www/another/app/static/;
        root /home/master/www/another/app;
        expires 31d;
    }
        
    #access_log /var/log/nginx/access.log clean;
    #error_log /var/log/nginx/error.log;
    
    #location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ {
    #    expires 30d;
    #}  
    #location ~ .*\.(js|css)?$ {
    #    expires 2h; 
    #}
}
```

#### nginx反向代理示例：

```nginx
server {
    listen [::]:80 ipv6only=off;

    server_name proxy.cherrot.com;

    location / {
            # rewrite ^/(.*) /$1 break;
            proxy_ignore_client_abort on;
            proxy_pass http://localhost:9200;
            # proxy_redirect http://localhost:9200 http://proxy.cherrot.com/;
            proxy_redirect default;
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header  Host $http_host;

            auth_basic "Need Authentication";
            auth_basic_user_file /home/master/data/user.pwd;
    }
}
```

可以看出，只要修改`proxy_pass`字段，一个限制特定域名的代理就完成了。
上面的配置用意是向公网提供代理访问某个监听`localhost:9200`的内网应用，并提供基本的安全限制（HTTP Authentication）因此使上述配置生效还需要生成一个用户密码文件，这项工作可以借助交互式终端程序`htpasswd`完成：

```bash
sudo apt-get install apache2-utils
htpasswd -c /home/master/data/user.pwd YourUserName
```

### For newbies: PHP+MySQL速配指南

#### 安装必要的软件包（可以根据个人需要精简部分php软件包——虽然没什么卵用）：

```bash
sudo apt-get install php5-cgi php5-mysql php5-fpm php5-curl php5-gd php5-idn php-pear php5-imap php5-mcrypt php5-mhash php5-pspell php5-recode php5-sqlite php5-tidy php5-xmlrpc php5-xsl mysql-server
```

安装`mysql-server`期间会要求用户设置root密码。
安装PHP字节码(opcode)加速器`XCache`(不建议使用APC，频繁写操作时会存在诡异的问题)：


```bash
apt-get install php5-xcache
```

#### 配置PHP Fast CGI (`php-fpm`)
编辑文件`/etc/php5/fpm/php.ini`:

```ini
# 修改最大上传大小为 10M，两个位置，一个 upload_max_filesize 一个 post_max_size
upload_max_filesize = 10M
post_max_size = 10M
# 修改最大内存使用，默认是 128M，如果你内存较多又是专门跑 PHP 的可以适当加大，具体数值建议做压力测试。
memory_limit = 128M
# 修改最大执行时间，默认 30，同样如果是跑 PHP 的服务器可以适当加大。
max_execution_time = 30
```
编辑文件 `/etc/php5/fpm/pool.d/www.conf`

```ini
# 修改侦听路径为 UNIX socket 提高效率（Ubuntu 14.04已默认）
listen = /var/run/php5-fpm.sock
# 修改 process manager 最大请求数量限制以防服务器挂掉，默认该语句被注释掉(应该是不限制？) 一般 VPS 直接取消注释即可。
pm.max_requests = 500
```

#### 配置Mysql数据库

1.  连接数据库

    ```bash
    mysql -u root -p
    ```

2.  创建数据库、表

    ```mysql
    # 新建数据库
    create database `wordpress`;
    # 新建用户
    create user `wordpress` identified by 'YourStrongPassWordHereLoL';
    # 设置权限
    grant all privileges on wordpress.* to wordpress@localhost identified by 'YourStrongPassWordHereLoL';
    # 刷新权限表
    flush privileges;
    # 退出
    exit
    ```

## 参考资料
* [Ubuntu 服务器配置简易指南][phoenix]
* [How To Set Up a Private Git Server on a VPS](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-git-server-on-a-vps)
* [Nginx HttpCoreModule#listen][nginx]


[phoenix]:http://blog.phoenixlzx.com/2014/02/01/simple-steps-with-ubuntu-server/
[nginx]:http://wiki.nginx.org/HttpCoreModule#listen
