---
title: HomeLab系列-Proxmox优化配置
tags:
  - proxmox
toc: true
categories:
  - HomeLab
date: 2024-06-05 19:25:44
---

安装了pve之后, 执行apt update的时候, 会提示订阅的问题, 导致无法update, 按照以下方法重新配置Apt源,Ceph源以及Pve中的订阅提示之后, 就不会有这些麻烦问题了!
<!--more-->

## Apt源配置
编辑/etc/apt/sources.list文件, 写入如下
```shell
deb https://mirrors.huaweicloud.com/debian/ bookworm main non-free contrib
deb-src https://mirrors.huaweicloud.com/debian/ bookworm main non-free contrib
deb https://mirrors.huaweicloud.com/debian-security/ bookworm-security main
deb-src https://mirrors.huaweicloud.com/debian-security/ bookworm-security main
deb https://mirrors.huaweicloud.com/debian/ bookworm-updates main non-free contrib
deb-src https://mirrors.huaweicloud.com/debian/ bookworm-updates main non-free contrib
deb https://mirrors.huaweicloud.com/debian/ bookworm-backports main non-free contrib
deb-src https://mirrors.huaweicloud.com/debian/ bookworm-backports main non-free contrib
```
## Ceph订阅配置
编辑/etc/apt/sources.list.d/ceph.list, 写入如下
```shell
deb https://mirrors.ustc.edu.cn/proxmox/debian/ceph-quincy bookworm no-subscription
```
## Pve安装源配置
编辑/etc/apt/sources.list.d/pve-install-repo.list, 写入如下
```shell
deb http://mirrors.ustc.edu.cn/proxmox/debian/pve bookworm pve-no-subscription
```
## 去除Pve订阅提示
执行以下命令即可
```shell
sed -i_orig "s/data.status === 'Active'/true/g" /usr/share/pve-manager/js/pvemanagerlib.js
sed -i_orig "s/if (res === null || res === undefined || \!res || res/if(/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
sed -i_orig "s/.data.status.toLowerCase() !== 'active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```
