---
title: 解决Push Image To Harbor报错unknow blob
date: 2024-04-08 22:55:49
tags: ['harbor', 'nginx']
categories: ['问题处理']
---
反向代理harbor后, 使用docker push时报错unknow blob

<!--more-->

## 现象

当对harbor进行反向代理后, 使用docker push推送镜像的时候, 报错unknow blob, 如下

![微信截图_20240408225934.png](https://mys3.kengdie.xyz/blog/微信截图_20240408225934.png)

查询后端harbor时, 发现报错blob unknow to registry, 如下

![微信截图_20240408230122.png](https://mys3.kengdie.xyz/blog/微信截图_20240408230122.png)

## 解决方法

类似问题见issue: https://github.com/goharbor/harbor-helm/issues/174, 如下

![微信截图_20240408230406.png](https://mys3.kengdie.xyz/blog/微信截图_20240408230406.png)

因为本次用的Harbor是使用OpenResty进行代理，后端Harbor本身采用http，并未使用ssl，前端OpenResty使用harbor.kengdie.xyz域名，并设置ssl

因此在代理配置中添加配置如下，即可

``` shell
proxy_set_header X-Forwarded-Proto "https";
```

![微信截图_20240408230509.png](https://mys3.kengdie.xyz/blog/微信截图_20240408230509.png)

> 添加client_max_body_size主要是为了避免推送大镜像时，报错Entity Too Large
> ![微信截图_20240408230559.png](https://mys3.kengdie.xyz/blog/微信截图_20240408230559.png)
> 添加参数后，再次推送镜像，即可成功，如下
> ![微信截图_20240408230635.png](https://mys3.kengdie.xyz/blog/微信截图_20240408230635.png)


