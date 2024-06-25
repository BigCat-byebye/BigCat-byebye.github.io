---
title: 沉默的yum指令
date: 2024-06-25 19:15:49
tags: ['linux', 'yum']
categories: ['问题处理']
toc: true
---


同事在重新部署tbase的时候, 需要安装一些rpm包, 但是`yum install`的时候, 莫名其妙的没有反应, 开始以为是源的问题, 

<!--more-->

## 现象

执行yum命令的时候, 没有任何输出, 直接结束, 甚至于执行`yum --help`的时候也是没有任何输出

## 排查

### 检查yum源

一开始以为是yum源的问题, 因为之前因为漏洞整改, 当时是记得关闭了yum源的, 就直接停止了yum的进程, 重新使用SimpleHTTPServer起了个yum之后

```shell
nohup python -m SimpleHTTPServer 8081 > /dev/null 2>&1 &
```

使用`yum repolist`执行看不到任何仓库, 直接执行yum也没有反应

### 排查是否被入侵

真的是奇怪到家了, 之所以想到入侵, 是因为, 在重新部署tbase之前, 是因为之前的tbase环境被破坏了, 当时找不到是谁破坏的, 因此选择的直接重新部署, 结果重新部署居然yum执行了, 没有任何输出, 想到了服务器被入侵也就毫不奇怪了.....

1. 查找文件路径

   ```shell
   whereis yum
   ls -lh /usr/bin/yum
   ```

   发现`/usr/bin/yum`是1个软连接, 指向`/usr/bin/dnf-3`这个文件

2. 对比文件md5

   ```shell
   md5sum /usr/bin/dnf-3
   md5sum /usr/bin/yum
   ```

   通过和正常执行yum的机器进行对比, 发现md5都是一样的

3. 对比python环境

   由于`/usr/bin/dnf-3`是个`python`文件, 而且是用的python3, 因此怀疑是不是python3被替换了导致的(**ps: 之所以这样想, 是因为以前我自己放在公网服务器的redis, 直接暴露了6379, 导致仅仅1天, 就被入侵了,  当时排查了好久, 后来是对比md5发现ls和cd这些命令被替换了, 所以, 不管怎么杀死进程, 那个进程都会起来, 所以, 对这种命令替换的入侵, 记忆特别深刻**), 因为系统上没有pip命令, 因此简单的通过对比python3的rpm包来确认是否是一样多

   ``` shell
   rpm -qa | grep python3 | wc -l
   ```

   经过对比, 包的数量和名称都是和正常机器一样的

### 排查yum执行过程

linux中有个`strace`命令可以查看命令或者进程的执行过程以及调用的函数

```shell
strace yum install vim -y
```

终于发现了蛛丝马迹

![](https://mys3.kengdie.xyz/blog/202406251952836.png)

可以看到显示磁盘不足

### 排查磁盘

直接`df -lh `可以看到`/var/log`磁盘已经到达了100%了, 到这里, 就排查结束了, **yum命令执行的时候, 会写日志, 但是磁盘满了, 写不了, 程序压根没有执行完, 因此根本就看不到任何输出**

![](https://mys3.kengdie.xyz/blog/202406251954344.png)

## 总结

1. 出现问题时, 首先第一应该排查的就是cpu,mem以及disk的使用情况, 这样的可以避免很多奇奇怪怪的问题, 同时也可以加快排查问题的速度.
