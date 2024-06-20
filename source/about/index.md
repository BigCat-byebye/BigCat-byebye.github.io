---
title: 关于我
toc: true
date: 2024-06-20 14:55:01
---

喜欢折腾的运维, 什么都喜欢搞一点的运维一枚哦!!!

# 公益资源

说明: 以下资源, 均为公益性质, 请手下留情! 具体使用方式, 看本站文章!

## Docker镜像站

| 源站                 | 镜像站                        |
| -------------------- | ----------------------------- |
| docker.io            | docker.kengdie.xyz            |
| quay.io              | quay.kengdie.xyz              |
| gcr.io               | gcr.kengdie.xyz               |
| k8s.gcr.io           | k8s-gcr.kengdie.xyz           |
| registry.k8s.io      | registry-k8s.kengdie.xyz      |
| ghcr.io              | ghcr.kengdie.xyz              |
| docker.cloudsmith.io | docker-cloudsmith.kengdie.xyz |

### 配置

配置Containerd镜像仓库

```shell
mirrors:
  "docker.io":
    endpoint:
      - "https://docker.kengdie.xyz"
```

### 使用

```shell
docker pull busybox
```

## API代理

```shell
https://apiproxy.kengdie.xyz/
```

### 使用

直接访问```https://apiproxy.kengdie.xyz/```加上```https://www.baidu.com``即可



