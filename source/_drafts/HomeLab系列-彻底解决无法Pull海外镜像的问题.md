---
title: HomeLab系列-彻底解决无法Pull海外镜像的问题
tags: ['proxmox', 'tailscale']
toc: true
categories: ['HomeLab']
---
因为天朝众所周知的问题, 拉取海外镜像经常会失败, 前几天买了个HK的机器,一下子搞了5年......正好可以用来做个镜像仓库的代理,可以用来中转dockerhub和其他的
<!--more-->

## 参考
https://distribution.github.io/distribution/recipes/mirror/
