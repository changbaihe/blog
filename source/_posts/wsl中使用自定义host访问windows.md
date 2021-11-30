---
title: wsl中使用自定义host访问windows
date: 2021-10-19 19:26:31
tags:
  - wsl
categories:
  - [windows]
  - [ubuntu]
---

话不多说，先献祭出，wsl官方文档：[[->(*^_^*)<-](https://docs.microsoft.com/zh-cn/windows/wsl/)]，对wsl没有一个完整了解的，可以先看看官方文档，对wsl有个整体的了解。

## 我与wsl的那点事

起初没用wsl之前，wins上装了SSR代理，一切都很和平，使用了wsl之后，在wsl之中git访问github，发现好慢，这个时候自然而然的想着去找~/目录下的.gitconfig，给github配置一下socks5代理，这个时候问题就来了，wins的ip从哪来？

## 探坑过程

大概整理情况如下：
- wsl和wins，是通过一个叫做 vEthernet(WSL) 的虚拟网络适配器，建立的局域网进行通信的。
- 理论上我们只需要，获取到 wins 在这个网络中的 ip，在wsl中写死就可以了，但是搞人的事情就发生了。。。
- 每次开机这个 vEthernet(WSL) 都会被重新创建，貌似是关机的时候会删掉，导致 我们上面拿到的ip是一直变的，他不是固定的，只要关开机，就一定会变
- 查询[wsl官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/networking#accessing-windows-networking-apps-from-linux-host-ip), 可以同过查询/etc/resolv.conf的nameserver，来确定wins ip

找到这，基本上就已经结果很明朗了。

思路如下:
- 直接搞一个查询当前wins ip并更新到/etc/hosts中的脚本
- 每次进入shell的时候，自动执行
- 这样我们在wsl直接使用 自定义域名(这块我叫的是winhost)就行了。

接下来，就开搞开搞~~~

## 具体步骤

### 给/etc/hosts 加上写权限

- 因为需要修改/etc/hosts，这个是我觉得不太好的地方，但是就是想映射自定义host, 暂时没想到别的办法
```bash
sudo chmod a+w /etc/hosts
```

### 获取wins ip并写入或更新/etc/hosts

- 来到~/目录下，为了方便以后复制，新建一个.bash_config.sh文件
``` bash
$ cd ~
$ vim .bash_config.sh
```

- 贴入如下代码：
```bash
# 自定义的host
winhost="winhost"
hostmap=$(echo $(ip route | grep default | awk '{print $3}') $winhost)
echo $hostmap
content=$(cat /etc/hosts | grep $winhost)
if [ "$content" ]
then
  cat /etc/hosts > /tmp/tmpwinhost
  sed -i "s/$content/$hostmap/g" /tmp/tmpwinhost
  cat /tmp/tmpwinhost > /etc/hosts
else
  echo -e "\n$hostmap" >> /etc/hosts
fi
```

- 给.bash_config.sh文件上 执行权限:
```bash
sudo chmod +x ~/.bash_config.sh
```

### 每次进入shell，自动执行 ~/.bash_config.sh

- 将.bash_config.sh 放到 .bashrc中，每次登录shell都自动更新wins的ip到 /etc/hosts
```bash
echo -e "\n\. ~/.bash_config.sh" >> ~/.bashrc
```