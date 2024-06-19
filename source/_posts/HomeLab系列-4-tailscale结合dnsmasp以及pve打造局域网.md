---
title: HomeLab系列-4-tailscale结合dnsmasp以及pve打造局域网
tags: ["pve", "tailscale", "dnsmasq"]
toc: true
categories: ['HomeLab']
date: 2024-06-19 22:51:00
---

 使用tailscale可以搭建内网环境, 可以实现pve上的虚拟机自动接入tailscale的内网, 且从远程的笔记本直接访问, 无需其他设置

## 参考

[Tailscale官网下载]([Download · Tailscale](https://tailscale.com/download))

[Tailscale Exit Node]([Exit nodes (route all traffic) · Tailscale Docs](https://tailscale.com/kb/1103/exit-nodes))

[Tailscale Subnet]([Subnet routers and traffic relay nodes · Tailscale Docs](https://tailscale.com/kb/1019/subnets))

[Tailscale DNS设置]([DNS - Tailscale](https://login.tailscale.com/admin/dns))

[Arch Wiki上的Dnsmasq配置]([dnsmasq - Arch Linux 中文维基 (archlinuxcn.org)](https://wiki.archlinuxcn.org/wiki/Dnsmasq))

## 在Pve上配置Tailscale的subnet

在pve的宿主机上安装好tailscale, 然后开启subnet功能

安装tailscale, 执行以下命令即可, 命令会打出tailscale的登录链接, 直接点进去登录即可认证成功

```shell
curl -fsSL https://tailscale.com/install.sh | sh
```

开启ipv4和ipv6转发

```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

如果你没有关闭firewalld的话, 还需要开启伪装, 执行如下, 如果关闭了firewalld的话, 就不用执行了

```shell
firewall-cmd --permanent --add-masquerade
```

开启tailscale, 如下

```shell
tailscale up --advertise-routes 10.10.10.0/24  --accept-routes
```

界面登录Tailscale, 选择刚才加入的节点, 选择如图的Disable key expirity, 修改设备为不过期, 因为间隔180天后,Tailscale默认会让加入的节点重新认证, 从而保证安全, 因为我的这些设备都是固定的, 因此直接关闭key过期

![image-20240611155354309](https://mys3.kengdie.xyz/blog/image-20240611155354309.png)

选择和Edit route settings, 勾选**设备**上报的子网, 那么该子网下的虚拟机都可以通过**设备**接入到Tailscale的内网, 而不用安装Tailscale客户端, 当然这个性能是没有安装客户端的性能好, 但是这样子更方便, **因为我就可以在本地直接通过内网ip连接pve创建出来的虚拟机**, 而不用每个虚拟机都安装Tailscale客户端了

![image-20240611155239016](https://mys3.kengdie.xyz/blog/image-20240611155239016.png)

## 在Vps上配置Tailscale的exit node

在**香港的vps**上装上tailscale, 然后开启exit-node功能, 这样子, 就可以让整个Tailscale的内网流量通过这个节点出去, 比如**现在dockerhub不是被封了么, 如果我将Tailscale的流量都从香港的vps走, 那么在上面建立一个镜像代理, 就可以通过这镜像代理拉取dockerhub的镜像了**!此处应该聪明的人可以想到别的用处吧? 对, 就是FQ

安装tailscale, 执行以下命令即可, 命令会打出tailscale的登录链接, 直接点进去登录即可认证成功

```shell
curl -fsSL https://tailscale.com/install.sh | sh
```

开启ipv4和ipv6转发

```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

如果你没有关闭firewalld的话, 还需要开启伪装, 执行如下, 如果关闭了firewalld的话, 就不用执行了

```shell
firewall-cmd --permanent --add-masquerade
```

开启tailscale, 如下

```shell
tailscale up  --advertise-exit-node=true --accept-routes=true  --accept-dns=true
```

界面登录Tailscale, 选择刚才加入的节点, 选择如图的Disable key expirity, 修改设备为不过期

修改Edit routes settings, 选择Use as exit node即可

![image-20240611164013741](https://mys3.kengdie.xyz/blog/image-20240611164013741.png)

## 在Pve上创建Sdn

### 创建区域

选择数据中心>SDN, 在区域这里添加1个区域, 类别的话选择simple即可, 具体可有参照下面的图

![image-20240611164827278](https://mys3.kengdie.xyz/blog/image-20240611164827278.png)

### 创建Vnet

区域创建完之后, 在vnets这里创建1个vnet就好, 区域的话, 选择刚才创建的区域, 其他的可以不用写, 具体如下图就好![image-20240611164935756](https://mys3.kengdie.xyz/blog/image-20240611164935756.png)

### 创建子网

创建了vnet之后, 点击子网, 在右边创建子网, 填写你要设置的网段, **这里的网段信息要注意, 最好选择1个和平时常用的网段不相同的区域, 比如和常用的192.168段或者172.16段区分开, 我这里选择的是10.10.10.0/24段, 同时需要注意下, 这个网段是要和上面的tailscale的网段对应的, 只有这样, 才能将pve上创建的vm加入到tailsacle的内网中来**, 这里的Dns前缀写tailscale中的内网前缀

![image-20240611165001632](https://mys3.kengdie.xyz/blog/image-20240611165001632.png)

### 重载网络

上述的步骤走完了之后, 切记, 一定要在SDN这里点击下应用, 也就是重载网络, 只有这样, 刚才新建的sdn这些才会生效, 然后创建虚拟机的时候, 才可以选择刚才创建的sdn

![image-20240611171539630](https://mys3.kengdie.xyz/blog/image-20240611171539630.png)

## 创建虚拟机验证

创建虚拟机, 选择刚才创建的sdn, 如下

![image-20240611171830100](https://mys3.kengdie.xyz/blog/image-20240611171830100.png)

登录后, 可以看到能够获取到正常的ip, 也就是10.10.10.14

![image-20240611172138811](https://mys3.kengdie.xyz/blog/image-20240611172138811.png)

## 修改Dnsmasq配置

在pve中创建sdn, 本质上就是pve会安装1个dnsmasq, 然后根据设置sdn生成1个专门的service和配置, 如下图中的dnsmasq@mysdn.service, 其中mysdn是上面创建的sdn, 其配置目录在/etc/dnsmasq.d/mysdn/下面

![image-20240611172411804](https://mys3.kengdie.xyz/blog/image-20240611172411804.png)

在其中的配置文件10-mynet10.conf中加入tailscale的内网的dns前缀, 其中的local和domain表示自定义域, 这个是tailscale的dns中定义的自定义域, dhcp-option表示给子网内的主机的网关为100.100.100.100, 这个是tailscale的dns中配置nameserver, 这样子, 相当于给子网内的所有主机的/etc/resolve.conf中都配置上1个search值以及nameserver

## 验证Dnsmasq

重启虚拟机后, 可以看到虚拟机获取到的ip中的nameserver和search值, 如下所示

![image-20240611220609690](https://mys3.kengdie.xyz/blog/image-20240611220609690.png)

这样子, 就实现了, 在虚拟机内部联通tailscale内网中的其他设备, 而不需要添加后缀了, 如下

![image-20240611221336531](https://mys3.kengdie.xyz/blog/image-20240611221336531.png)

从笔记本到虚拟机是可以直接ping通的

![image-20240611221241433](https://mys3.kengdie.xyz/blog/image-20240611221241433.png)
