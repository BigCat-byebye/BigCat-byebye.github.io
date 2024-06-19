---
title: 脚本-用Excel管理SSH主机
date: 2024-05-10 17:10:06
tags: ['bash', '脚本', 'ssh']
toc: true
categories: ['脚本工具']
---

最简单的SSH主机管理方式
- 使用Excel表格管理ssh主机登录信息
- 使用Windows Terminal中的ssh cliet登录ssh

<!--more-->


## 要求

- 文件为csv格式,保存为默认编码(**ANSI格式**)或者**新建1个Excel另存为csv**即可
- **密钥文件, csv文件以及脚本文件必须处于同一目录**
- 使用密钥登录的话, **密钥文件必须以pem和key结尾**

## Csv文件

新建1个Excel, 列名分别为name, ip, port, user, password即可, 填写完后保存为csv文件

- 字段顺序不可以更改, 必须为name, ip, port, user, password
- password字段填写密码或者密钥文件名, 只写密钥文件名即可, 且必须放在和csv文件同一目录

![link-to-ssh-csv](https://mys3.kengdie.xyz/blog/link-to-ssh-csv.png)

## Bash脚本

```shell
➜  ~ cat link-to-ssh.sh
#!/bin/bash
#set -exu

# 函数：显示选择列表并返回用户选择的选项
# 参数：
#   $1 - 选项数组
# 返回值：
#   用户选择的选项
# 定义选择函数
select_option() {
  choices=("$@")  # 将选项数组声明为全局变量
  selected=0      # 初始化选择索引

  while true; do
    clear
    for index in "${!choices[@]}"; do
      if [ $index -eq $selected ]; then
        printf "\033[31m> ${choices[$index]}\033[0m\n"  # 高亮显示选中的选项
      else
        echo "  ${choices[$index]}"
      fi
    done

    read -n1 -s key  # 读取单个按键并保持输入的隐私

    case "$key" in
      A)  # 上箭头
        if [ $selected -gt 0 ]; then
          selected=$((selected - 1))
        fi
        ;;
      B)  # 下箭头
        if [ $selected -lt $(( ${#choices[@]} - 1 )) ]; then
          selected=$((selected + 1))
        fi
        ;;
      "")  # 回车键
        break
        ;;
    esac
  done

  # 打印最终结果日志
  selected_option="${choices[$selected]}"
  echo "最终选择：$selected_option"
}

isReadlink=$(readlink $0 | wc -l)

if [ $isReadlink == "0" ]
then
        echo -e "\e[31m警告: $0只能是1个软连接\e[0m"
        exit 999
fi

apt install -y sshpass > /dev/null 2>&1

sshFile=$(readlink $0 | sed "s@link-to-ssh.sh@link-to-ssh.csv@g")

if [ $( ls $sshFile | wc -l) == "0" ]
then
        echo -e "\e[31m警告: link-to-ssh.sh和link-to-ssh.csv不在同一目录\e[0m"
        exit 999
fi

options=()
index=0
for line in `iconv -f GBK -t UTF-8 $sshFile | grep -v name`
do
        info=$(echo $line | awk -F ',' '{print $1}')
        options[$index]=$info
        index=$((index+1))
done


select_option "${options[@]}"

sshIp=$(iconv -f GBK -t UTF-8 $sshFile | grep $selected_option | awk -F',' '{print $2}')
sshPort=$(iconv -f GBK -t UTF-8 $sshFile | grep $selected_option | awk -F',' '{print $3}')
sshUser=$(iconv -f GBK -t UTF-8 $sshFile | grep $selected_option | awk -F',' '{print $4}')
sshPassword=$(iconv -f GBK -t UTF-8 $sshFile | grep $selected_option | awk -F',' '{print $5}')

endWith=$(echo $sshPassword | grep -E "pem$|key$" |wc -l)

if [ $endWith == "1" ]
then
        sshKeyFile=$(readlink $0 | sed "s@link-to-ssh.sh@$sshPassword@g")

        if [ $( ls $sshKeyFile | wc -l) == "0" ]
        then
                echo -e "\e[31m警告: link-to-ssh.sh和link-to-ssh.csv以及ssh密钥文件不在同一目录\e[0m"
                exit 999
        fi

        cp -rf $sshKeyFile /tmp/ && chmod 600 /tmp/$sshPassword
        nohup sleep 5 && rm -rf /tmp/$sshPassword > /dev/null 2>&1 &
        ssh -i /tmp/$sshPassword -p $sshPort -o StrictHostKeyChecking=no -o ServerAliveInterval=60 -o ServerAliveCountMax=30 $sshUser@$sshIp
else
        sshpass -p $sshPassword ssh -p $sshPort -o StrictHostKeyChecking=no -o ServerAliveInterval=60 -o ServerAliveCountMax=30 $sshUser@$sshIp
fi
➜  ~
```


## 运行

新建1个软链接

```shell
ln -s /xxx/yyy/link-to-ssh.sh .
```

运行

```shell
bash link-to-ssh.sh
```

如下

![link-to-ssh-sh-running](https://mys3.kengdie.xyz/blog/link-to-ssh-sh-running.png)