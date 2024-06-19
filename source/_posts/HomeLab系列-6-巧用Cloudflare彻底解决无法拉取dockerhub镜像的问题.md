---
title: HomeLab系列-6-巧用Cloudflare彻底解决无法拉取dockerhub镜像的问题
tags:
  - proxmox
  - tailscale
toc: true
categories:
  - HomeLab
  - Cloudflare
date: 2024-06-17 12:36:50
---

前几天写了个使用registry的pullthrough功能,解决了无法拉取dockerhub的问题, 但是最近又碰到了其他问题, 那就是在拉取镜像的时候, yaml文件中的镜像不一定都是dockerhub中的, 而且, 用registry还有单独解决https的问题, 如果只解决dockerhub还好, 但是镜像仓库多了就很麻烦, 因此想到了cloudflare的worker, 果然能完美解决这个问题!!!
<!--more-->

## 参考
[ciiiii/cloudflare-docker-proxy: A docker registry proxy run on cloudflare worker. (github.com)](https://github.com/ciiiii/cloudflare-docker-proxy)

## 使用方法-无敌简单版

直接进入参考的github网页, 在仓库中直接点击这个按钮即可
![image-20240617124138371](https://mys3.kengdie.xyz/blog/image-20240617124138371.png)

然后会跳出来github的认证, 再填写上cloudflare的token即可

## 使用方法-手动版
因为需要创建token, 然后不太熟悉cloudflare的token, 但是如果给个全局token, 又感觉太危险, 索性手撸了, 不用他那个自动化了, 他那个自动化, 就是把worker自动部署一下, 本质上来说就是自动了而已

### 托管网站

准备1个自己的域名, 在cloudflare的网站这里点击添加站点, 根据提示, 到**域名的服务商**那里将**dns服务器**改成**提示**中的

说明: 域名服务商,就是你在哪里申请的域名, 哪里就是你的域名服务商, 一般你申请了域名之后, 会有1个管理后台的, 管理后台中, 就有域名的**dns服务器**, 你改成**提示**中的cloudflare的就好, 一般需要数小时才能识别到, 等cloudflare识别了之后, 这里的站点下面, 就会出现活跃2个字, 就可以进行下一步了

![image-20240617125118693](https://mys3.kengdie.xyz/blog/image-20240617125118693.png)

### 创建worker

登录到cloudflare的界面, 然后点击Workers页面, 如下, 不熟悉英文的人, 可以选择右上角, 选择中文就是了

![image-20240617124527386](https://mys3.kengdie.xyz/blog/image-20240617124527386.png)

创建1个worker, 名字的话, 就写cloudflare-docker-proxy就可以了, 因为这个名字被worker中的代码里面引用了, 因此就直接写这个名字吧

### 编辑代码

注意修改其中的routes中的域名地址为自己的, 也就是下面的yyy.com替换为自己的域名, 建议直接全局替换即可, 记住这里的域名分别是

```shel
docker.yyy.xyz对应dockerHub也就是docker.io
quay.yyy.xyz对应https://quay.io
gcr.yyy.xyz对应https://gcr.io
k8s-gcr.yyy.xyz对应https://k8s.gcr.io
registry-k8s.yyy.xyz对应https://registry.k8s.io
ghcr.yyy.xyz对应https://ghcr.io
docker-cloudsmith.yyy.xyz对应https://docker.cloudsmith.io
```

worker的代码如下

```javascript
addEventListener("fetch", (event) => {
  event.passThroughOnException();
  event.respondWith(handleRequest(event.request));
});

const MODE = "production";
const TARGET_UPSTREAM = "";

const dockerHub = "https://registry-1.docker.io";

const routes = {
  // production
  "docker.yyy.xyz": dockerHub,
  "quay.yyy.xyz": "https://quay.io",
  "gcr.yyy.xyz": "https://gcr.io",
  "k8s-gcr.yyy.xyz": "https://k8s.gcr.io",
  "registry-k8s.yyy.xyz": "https://registry.k8s.io",
  "ghcr.yyy.xyz": "https://ghcr.io",
  "docker-cloudsmith.yyy.xyz": "https://docker.cloudsmith.io",

  // staging
  // "docker-staging.yyy.xyz": dockerHub,
};

function routeByHosts(host) {
  if (host in routes) {
    return routes[host];
  }
  if (MODE == "debug") {
    return TARGET_UPSTREAM;
  }
  return "";
}

async function handleRequest(request) {
  const url = new URL(request.url);
  const upstream = routeByHosts(url.hostname);
  if (upstream === "") {
    return new Response(
      JSON.stringify({
        routes: routes,
      }),
      {
        status: 404,
      }
    );
  }
  const isDockerHub = upstream == dockerHub;
  const authorization = request.headers.get("Authorization");
  if (url.pathname == "/v2/") {
    const newUrl = new URL(upstream + "/v2/");
    const headers = new Headers();
    if (authorization) {
      headers.set("Authorization", authorization);
    }
    // check if need to authenticate
    const resp = await fetch(newUrl.toString(), {
      method: "GET",
      headers: headers,
      redirect: "follow",
    });
    if (resp.status === 401) {
      if (MODE == "debug") {
        headers.set(
          "Www-Authenticate",
          `Bearer realm="http://${url.host}/v2/auth",service="cloudflare-docker-proxy"`
        );
      } else {
        headers.set(
          "Www-Authenticate",
          `Bearer realm="https://${url.hostname}/v2/auth",service="cloudflare-docker-proxy"`
        );
      }
      return new Response(JSON.stringify({ message: "UNAUTHORIZED" }), {
        status: 401,
        headers: headers,
      });
    } else {
      return resp;
    }
  }
  // get token
  if (url.pathname == "/v2/auth") {
    const newUrl = new URL(upstream + "/v2/");
    const resp = await fetch(newUrl.toString(), {
      method: "GET",
      redirect: "follow",
    });
    if (resp.status !== 401) {
      return resp;
    }
    const authenticateStr = resp.headers.get("WWW-Authenticate");
    if (authenticateStr === null) {
      return resp;
    }
    const wwwAuthenticate = parseAuthenticate(authenticateStr);
    let scope = url.searchParams.get("scope");
    // autocomplete repo part into scope for DockerHub library images
    // Example: repository:busybox:pull => repository:library/busybox:pull
    if (scope && isDockerHub) {
      let scopeParts = scope.split(":");
      if (scopeParts.length == 3 && !scopeParts[1].includes("/")) {
        scopeParts[1] = "library/" + scopeParts[1];
        scope = scopeParts.join(":");
      }
    }
    return await fetchToken(wwwAuthenticate, scope, authorization);
  }
  // redirect for DockerHub library images
  // Example: /v2/busybox/manifests/latest => /v2/library/busybox/manifests/latest
  if (isDockerHub) {
    const pathParts = url.pathname.split("/");
    if (pathParts.length == 5) {
      pathParts.splice(2, 0, "library");
      const redirectUrl = new URL(url);
      redirectUrl.pathname = pathParts.join("/");
      return Response.redirect(redirectUrl, 301);
    }
  }
  // foward requests
  const newUrl = new URL(upstream + url.pathname);
  const newReq = new Request(newUrl, {
    method: request.method,
    headers: request.headers,
    redirect: "follow",
  });
  return await fetch(newReq);
}

function parseAuthenticate(authenticateStr) {
  // sample: Bearer realm="https://auth.ipv6.docker.com/token",service="registry.docker.io"
  // match strings after =" and before "
  const re = /(?<=\=")(?:\\.|[^"\\])*(?=")/g;
  const matches = authenticateStr.match(re);
  if (matches == null || matches.length < 2) {
    throw new Error(`invalid Www-Authenticate Header: ${authenticateStr}`);
  }
  return {
    realm: matches[0],
    service: matches[1],
  };
}

async function fetchToken(wwwAuthenticate, scope, authorization) {
  const url = new URL(wwwAuthenticate.realm);
  if (wwwAuthenticate.service.length) {
    url.searchParams.set("service", wwwAuthenticate.service);
  }
  if (scope) {
    url.searchParams.set("scope", scope);
  }
  headers = new Headers();
  if (authorization) {
    headers.set("Authorization", authorization);
  }
  return await fetch(url, { method: "GET", headers: headers });
}
```

替换为自己的域名后, 直接部署即可

### 自定义域名

在worker的页面点击刚才新建好的cloudflare-docker-proxy, 在设置的触发器, 这里添加自定义域, 然后人别填写你自己域名的子域名即可, 就是上面替换后的域名, 

```shell
docker.yyy.xyz
quay.yyy.xyz
gcr.yyy.xyz
k8s-gcr.yyy.xyz
ghcr.yyy.xyz
docker-cloudsmith.yyy.xyz
```

创建后如下

![image-20240617125632811](https://mys3.kengdie.xyz/blog/image-20240617125632811.png)

创建后, 就会出现6个自定义域

## 验证

### 登录dockerhub验证

使用docker login docker.yyy.com可以看到是能正常代理到dockers.io, 当然了, 账号密码用你在dockerhub的账号密码

```shell
docker login docker.yyy.com -u 23523523 -p 232523525
```

可以看到, 能够登录成功, 也就是说明正常代理到了dockerhub了

![image-20240617130259130](https://mys3.kengdie.xyz/blog/image-20240617130259130.png)

### 拉取dockerhub镜像验证

拉取busybox的镜像

```shell
docker pull docker.yyy.com/library/busybox:latest
```

可以看到能够正常拉取

![image-20240617130603461](https://mys3.kengdie.xyz/blog/image-20240617130603461.png)

### 拉取quay镜像验证

拉取镜像如下

```shell
docker pull quay.yyy.com/brancz/kube-rbac-proxy:v0.14.2
```

可以看到能正常拉取

![image-20240617130905600](https://mys3.kengdie.xyz/blog/image-20240617130905600.png)

可以看到均通过cloudflare正常代理了