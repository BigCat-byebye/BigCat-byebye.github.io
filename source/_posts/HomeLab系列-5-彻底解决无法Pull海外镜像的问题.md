---
title: HomeLab系列-5-彻底解决无法Pull海外镜像的问题
tags:
  - proxmox
  - tailscale
toc: true
categories:
  - HomeLab
date: 2024-06-10 17:26:59
---

因为天朝众所周知的问题, 拉取海外镜像经常会失败, 前几天买了个HK的机器,一下子搞了5年,再加上最近好像国内的dockerhub的镜像仓库被封了......正好可以用来做个镜像仓库的代理,可以用来中转dockerhub和其他的国外镜像仓库
<!--more-->

## 参考
https://distribution.github.io/distribution/recipes/mirror/

## 配置
写个配置文件如下, 注意其中username和password这2个哦, 本质上来说, registry只是替你去dockerhub拉取镜像而已,不是凭空造一个镜像,因此它去dockerhub拉镜像用的账号密码都是你的
```shell
[root@forward-hk-01 registry]# cat config-dockerhub.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
proxy:
  remoteurl: https://registry-1.docker.io
  username: 账号写自己登录dockerhub的账号
  password: 密码写自己登录dockerhub的密码
  ttl: 168h
delete:
  enabled: true
```

## 运行脚本
写一个bash脚本如下
```shell
[root@forward-hk-01 registry]# cat run-registry-dockerhub.sh
#!/bin/bash
docker rm -f registry-dockerhub
docker run -d -p 5000:5000 -v ./config-dockerhub.yml:/etc/docker/registry/config.yml  --name registry-dockerhub registry:2
```
写完直接
```shell
bash run-registry-dockerhub.sh
```
如下
```shell
[root@forward-hk-01 registry]# docker ps
CONTAINER ID  IMAGE                         COMMAND               CREATED     STATUS      PORTS                   NAMES
9f88e4faa92d  docker.io/library/registry:2  /etc/docker/regis...  2 days ago  Up 4 hours  0.0.0.0:5000->5000/tcp  registry-dockerhub
```

## 验证
### 配置docker仓库
在本机的/etc/docker/daemon.json中写入如下
```shell
➜  ~ cat /etc/docker/daemon.json
{
  "registry-mirrors": ["http://forward-hk-01:5000"],
  "insecure-registries": ["forward-hk-01:5000"]
}
➜  ~
```
这里说明下哈, forward-hk-01是运行上面的registry的机器哈, 只是我做了域名解析哈, 用的是tailscale, 关于这一块后面会在**HomeLab系列**进行详细讲解

### 重启docker
其实不一定要重启, 使用reload也行
```shell
➜  ~ systemctl reload docker
```

### 拉取验证
在本地拉取镜像
```shell
➜  ~ docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
ec562eabd705: Pull complete
Digest: sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest
```
可以看到busybox镜像能否被正常拉取, 而且我也没有指定仓库, 默认就是从docker.io拉取的, 但是可以正常拉取到

对比查看registry-dockerhub的日志如下
```shell
^C[root@forward-hk-01 registry]# docker logs --tail 5 -f  registry-dockerhub
10.88.0.1 - - [10/Jun/2024:07:26:36 +0000] "POST /sdk HTTP/1.1" 404 19 "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36"
10.88.0.1 - - [10/Jun/2024:07:26:37 +0000] "GET /evox/about HTTP/1.1" 404 19 "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36"
10.88.0.1 - - [10/Jun/2024:07:27:02 +0000] "GET / HTTP/1.0" 200 0 "" ""
10.88.0.1 - - [10/Jun/2024:07:27:02 +0000] "GET / HTTP/1.1" 200 0 "" ""
10.88.0.1 - - [10/Jun/2024:09:15:32 +0000] "GET / HTTP/1.1" 200 0 "" "Expanse, a Palo Alto Networks company, searches across the global IPv4 space multiple times per day to identify customers&#39; presences on the Internet. If you would like to be excluded from our scans, please send IP addresses/domains to: scaninfo@paloaltonetworks.com"



10.88.0.1 - - [10/Jun/2024:09:22:53 +0000] "GET /v2/ HTTP/1.1" 200 2 "" "docker/20.10.25 go/go1.18.1 git-commit/20.10.25-0ubuntu1~22.04.1 kernel/5.15.146.1-microsoft-standard-WSL2 os/linux arch/amd64 UpstreamClient(Docker-Client/20.10.25 \\(linux\\))"
time="2024-06-10T09:22:53.251386706Z" level=info msg="response completed" go.version=go1.20.8 http.request.host="forward-hk-01:5000" http.request.id=58510054-e56e-4cd3-943e-4dd94e8bb6a1 http.request.method=GET http.request.remoteaddr="10.88.0.1:60770" http.request.uri="/v2/" http.request.useragent="docker/20.10.25 go/go1.18.1 git-commit/20.10.25-0ubuntu1~22.04.1 kernel/5.15.146.1-microsoft-standard-WSL2 os/linux arch/amd64 UpstreamClient(Docker-Client/20.10.25 \(linux\))" http.response.contenttype="application/json; charset=utf-8" http.response.duration=1.32511ms http.response.status=200 http.response.written=2
```
可以看到刚才拉取busybox的客户端的相关信息