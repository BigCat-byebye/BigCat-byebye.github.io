---
title: 脚本-Zerotier-one自动加入
date: 2024-04-08 01:11:23
tags: ['bash', '脚本', 'zerotier-one', '内网穿透']
toc: true
categories: ['脚本工具']

---

本脚本用于一键式将客户端加入到zerotier-one的私有网络中

<!--more-->

## Bash脚本
``` shell
#!/bin/bash
#set -exu

# 从官网获取api token和子网id，目前不变
apitoken="fydPNeCNcAZMSEaRPa1adhHYqrMCoklK"
networkid="b15644912ead2af7"
# 填写客户端的名字和描述
clientname="test-cloud"
clientdescription="测试"

# 检测并安装zerotier-one, curl, jq
function checkandinstall() {
    whereis curl > /dev/null 2>&1 || whereis jq > /dev/null 2>&1
    if [[ $? -ne 0 ]]; then
        echo "请先安装curl和jq" && exit 1
    fi
    systemctl status zerotier-one >/dev/null 2>&1
    if [[ $? -ne 0 ]]; then
        echo "接下来，将使用官网脚本安装zerotier-one" && sleep 2
        curl -s https://install.zerotier.com | sudo bash
        if [[ $? -eq 0 ]]; then
            echo "Zerotier-one安装成功" && return 0
        fi
    fi

}

checkandinstall

echo "申请加入网络" && /usr/sbin/zerotier-cli join $networkid > /dev/null 2>&1
if [[ $? -eq 0 ]]; then
    # 设置访问的api token
    headers="Authorization: token $apitoken"
    # 获取本机客户端id
    clientid=$(/usr/sbin/zerotier-cli info | awk '{print $3}')
    # 确认客户端id是否已认证
    authorized=$(curl -s -X GET -H "$headers" https://my.zerotier.com/api/network/$networkid/member/$clientid | jq -r ".config.authorized")

    # 如果该设备还没有经过验证，则发送验证请求
    if [[ $authorized == false ]]; then
        authorize_url="https://my.zerotier.com/api/network/$networkid/member/$clientid"
        data='{"config": {"authorized": true}, "name": "'$clientname'" , "description": "'$clientdescription'"}'
        # 对客户端进行认证
        echo "网络认证中" && curl -s -X POST -H "$headers" --data "$data" $authorize_url >/dev/null 2>&1 && echo "网络认证成功"
        # 确认客户端id是否已认证
        authorized=$(curl -s -X GET -H "$headers" https://my.zerotier.com/api/network/$networkid/member/$clientid | jq -r ".config.authorized")
        if [[ $authorized == true ]]; then
                echo "客户端已认证成功,Zerotier-one子网中所有设备信息如下"
                # 获取所有客户端信息
                curl -s -X GET -H "$headers" https://my.zerotier.com/api/network/$networkid/member | jq -r '.[] | [.nodeId, .online, .config.ipAssignments[],.name, .description] | join("\t\t")' | awk 'BEGIN{printf "%-15s\t%-15s\t%-20s%-15s%-15s\n","nodeid","online","ip","name","description"}{printf "%-15s\t%-15s\t%-20s%-15s%-15s\n", $1,$2,$3,$4,$5}'
        fi
    fi
fi
```

## 使用

### 依赖

需要事先安装curl以及jq命令

### 修改配置

1. apitoken: 来自官网
![apitoken](../images/image-1.png)

2. networkid: 来自官网
![networkid](../images/image.png)

3. clientname: 预加入的客户端名称

4. clientdescription: 预加入的客户端描述

### 运行方式
执行
``` shell
bash auto-join-zerotier.sh
```