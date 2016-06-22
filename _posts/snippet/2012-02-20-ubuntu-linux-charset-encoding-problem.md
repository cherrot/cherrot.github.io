---
layout: post
title: Linux 文件/文件名乱码问题总结
date: 2012-02-20 23:02:49
categories: snippet
tags: encoding
description: 

---
首先推荐 Ubuntu技巧总结 wiki: [http://wiki.ubuntu.org.cn/UbuntuSkills#.E4.B8.AD.E6.96.87](http://wiki.ubuntu.org.cn/UbuntuSkills#.E4.B8.AD.E6.96.87)

## 查看编码
经过了数十年的编码混战，`UTF-8`以它字符覆盖广、节约字节占用（牺牲了一定性能）、且二进制安全(8bit)的特性逐渐站稳脚跟成为主流。但毕竟还有很多异教徒的存在，常见异教徒就有`GB2312`,`GBK`(别名`cp936`),`GB18030`,`ISO8859-1`(别名`latin1`)等。

这些异教徒中，`GB`开头的字符集格外臭名昭著。而`ISO8859-1`其实是扩展ASCII编码，所以多是用来背黑锅的角色，比如一些解压程序会把GBK等编码的文件名全部按`ISO8859-1`处理。

### file 命令

`file 文件名` 测试结果：

```
$ file README
README: ASCII text
```

### enca 命令（需要安装enca）

`enca 文件名` 或者可以添加语言代码参数： `enca -L zh_CN 文件名` 测试结果：

```
$ enca README
7bit ASCII characters
```

### vim 编辑器中察看文件编码

`:set fileencoding`

## 解决文件名乱码

### 使用convmv（通用解）

`convmv -f GBK -t UTF-8 *.txt` 该方法将会测试所有的 .txt 文件名转码，如果确认无误，就在前一条命令中加上 --notest 参数执行文件名转码： `convmv -f GBK -t UTF-8 --notest *.txt` -f （from）是原文件名编码（一般是GBK、GB2312或GB18030）, -t （to）是目标文件名编码，设为UTF-8。 另外还可以使用 -r 参数递归处理子目录。


### zip压缩包
`unzip -O cp936` 可以指定解压编码(Ubuntu下unzip支持该参数)

## 解决文件内容乱码

### 使用iconv

`iconv -f GBK -t UTF-8 gbkFile > newFile` 将gbkFile由GBK编码转换为UTF-8编码，并将转码后的结果重定向输出到 newFile中。（>相当于新建newFile文件保存结果，若使用>>则表示追加文件内容） 或者 `iconv -f GBK -t UTF-8 gbkFile -o newFile` 作用相同。 Tips: 将当前目录下所有php文件（递归子目录）由GBK转为UTF-8（覆盖原文件）：

```
find . -type f -name *.php -print -exec iconv -f UTF-8 -t GBK {} -o {} \;
```


### 使用enca

`enconv -L zh_CN -x UTF-8 gbkFile > newFile` 作用同上。 -L 参数指定语言代码而非文件编码，比如zh_CN代表大陆简体，这也是enca智能的地方。 -x指定输出文件编码，加上>newFile使转码结果保存到newFile中。

### 使用Vim编辑器进行转码

使用命令： `:set fileencoding=utf-8` 将当前文件转换为utf-8编码
