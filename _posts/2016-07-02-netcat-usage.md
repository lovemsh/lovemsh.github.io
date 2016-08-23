---
layout: post
title:  "使用netcat与服务器间传输文件"
date:   2016-07-02 02:00:00
categories: technology
tags: linux
permalink: /archivers/netcat_usage
---

### 前言
换了mac本之后，一直使用item2作为终端，最常用的就是通过ssh连接到远程服务器进行一些开发调试的工作。有时候需要把本地文件上传到服务器上，由于中间使用了跳板机，所以无法通过scp命令直接把文件上传到服务器上，而是要通过跳板机周转一下，非常麻烦。

在windows中经常使用xshell、secretCRT，可以很方便的通过rz、lz命令就可以上传下载文件。可是在MAC的item2中我按照教程配置了好久，依然不行，每次都是卡死。

终于，在今天，我在同事那里获知了有“瑞士军刀”美誉的工具—— nc（netcat）。

### 初试牛刀
使用nc传送文件很简单，在服务器端用nc监听一个端口用于接收文件，然后在客户端使用nc访问服务器IP和相应端口传输文件就OK了。

```
#远程服务器，接收文件。其中20000是监听的端口号，1.txt是你要把接收的文件保存的位置
nc -l 20000 > 1.txt

#本地机器，发送文件。其中10.60.236.62是远程服务器的ip，端口号要和监听的端口一致
nc  10.10.10.10 20000  <  1.txt
```

这样就可以把本地的文件上传到远程服务器了，注意要先启动接收的服务器

### 简单升级
每次输入这些命令还是比较麻烦的，因此，我自己写了一个的wrap脚本，可以通过简单的命令使用nc。

- 本地工具脚步 

首先，新建一个文件/usr/local/bin/mync，添加如下代码

```
#!/bin/bash
#远程服务器的ip
ip=10.10.10.10
port=20000
if [ -f $1 ];then
    nc -n $ip $port < $1
    if [ $? -eq 0 ]; then
        echo 'send ok'
    fi
else
    echo "$1 is not exist"
fi
```

然后给予这个脚本执行权限， 同时保证PATH中包含/usr/local/bin目录

```
chmod a+x /usr/local/bin/mync
export PATH="/usr/local/bin:$PATH"
```

- 服务器端工具脚步

新建一个文件/usr/local/bin/ncrev，添加如下代码

```
#!/bin/bash

port=20000
if [ -z $1 ]; then
    echo "please enter file name"
    exit 1
fi
dname=$( dirname $1 )
#判断目录是否存在，不存在则创建
if [ ! -d $dname ]; then
    mkdir -p $dname
    if [ $? -ne 0 ]; then
        echo "create dir failed"
        exit 1
    fi
fi
#判断用于存储的文件是否存在，如果存在，询问用户是否覆盖
if [ -f $1 ]; then
    echo "file is exist,overwrite it ?(y/n)"
    read input
    if [ $input != "y" -a $input != "Y" ]; then
        exit 2
    fi
fi

if [ -d $1 ]; then
    echo 'path can not a directory'
    exit 3
fi
echo "receiving..."
#开始监听
nc -l $port > $1
echo "received success..."
```

然后给予这个脚步执行权限， 同时保证PATH中包含/usr/local/bin目录

```
chmod a+x /usr/local/bin/ncrev
export PATH="/usr/local/bin:$PATH"
```

使用方法：

```
#首先在服务器端开启接收命令
ncrev /data/1.txt

#在客户端执行发送命令
mync /home/1.txt
```


这样就把本地/home/1.txt文件的内容发送到10.10.10.10服务器上，并保存在了 /data/1.txt 中，有没有觉得方便一些了？

### 结语

做的比较简单，基本满足了自己的需求，后期会进一步完善一下。

当然，nc的功能绝不仅限于此，更多nc的使用请参考： <http://netcat.sourceforge.net/> 或自行google