---
title: 记录Ubuntu 20.04搭建OpenVPN服务端和客户端
date: 2021-11-23 22:30:42
tags:
- openvpn
categories:
- [os, ubuntu]
---
依旧是那个位置，依旧是那个笑脸，[[->(^_^)<-](https://ubuntu.com/server/docs/service-openvpn)] (ubuntu-server版openvpn原文档)，没错，我是“docs党”，同时也是“google党”和“baidu党”，哈哈哈~

## 我与OpenVPN的故事 (来篇小作文吧~)
- 节奏1：mac上用Tunnelblick，，，，简直恶心，，，版本问题好多，被恶心到好几次，然后对配置也不了解，有问题了也不知道咋解决，化身google党，baidu党，稀里糊涂凑合用，之后发现Viscosity这款软件不错，就很少出问题，虽然配置仍然不懂。。。这个时候已经有想去好好研究下的想法了。
- 节奏2：就在前不久，我赶着win11热乎发布，我重装凑合用了一年多的win10(之前觉得系统有点乱，也没找到一个合适点去重装系统，也是懒的重复折腾系统那点事)，确实凑合久了，装系统的时候很痛快的，直接一把all in —— 格式化C盘，事后，就顿时酸爽了，我的环境和配置忘整出来了，OpenVPN自然也没了。。。那就从新装吧，作为一个docs党嘛，自然是去找它的官网了，下载最新客户端(就是喜欢体验新版本)，，，，下载下来，使用原来的证书和配置似乎连不上。。。~[此处略过一段悲伤的记忆]~，记得当时证书一直报错啥tap啥的，看配置一时间又整不明白，毫无头绪。。。最后下载旧版2.3尝试，最后算是可以了。。。这个时候心里对它开始躁动了~
- 节奏3：时间来到了最近，整了一个二手的tower主机，我装了一个ubuntu-server镜像，放在屋里当服务器学习，一开始，路由器上配置了自己电脑的远程桌面的端口，想着在公司可以远程自己的电脑，省的带电脑。碰巧就遇到一个ssh的问题，想在自己服务器上试试ssh相关的操作。于是，我通过远程面屋里的电脑，登录到路由器配置页，开始配置端口，这个时候脑残的一幕出现了，手一哆嗦点到了删除所有端口映射。。。。当场歇菜。。。顿时蒙蔽。。。眼前千万个草泥马奔腾而过~~~又想笑又无奈，只能等晚上回去搞了。这个时候OpenVPN慢慢的在我的脑海中升起来了，，，就好像说，来吧是时候展现真正的技术了，把我搞了，这不随便整，配一次端口就行了，还要啥自行车呢。嗯~有道理，那就搞喽(我忍你很久了)~
- 行动：干它鸭的，开搞开搞~

## 搭建过程 (看着↑头提到的文档)

### 安装工具包和证书生成 (假设当前是root用户)
  - 安装openvpn 和证书工具easy-rsa
    - apt update 
    - apt install openvpn  // 默认目录为/etc/openvpn
    - apt install easy-rsa
  - 备份easy-rsa脚本 (软链接了一下当前全局的脚本，防止更新，差不多算锁版本吧)
    - make-cadir /etc/openvpn/easy-rsa
  - 切换到easy-rsa目录
    - cd /etc/openvpn/easy-rsa
  - 初始化pki目录和相关模板(/etc/openvpn/easy-rsa/pki)
    - ./easyrsa init-pki
  - 创建公共ca.cert (pki/ca.crt)
    - ./easyrsa build-ca
  - 生成dh算法参数 (pki/dh.pem)
    - ./easyrsa gen-dh
  - 生成server的私钥key和证书req，使用公共用ca.crt以及vars，签名生成server.crt
    - ./easyrsa gen-req [server-name] nopass
    - ./easyrsa sign-req server [server-name]
  - 将server参数复制到openvpn server主目录
    - cp pki/dh.pem pki/ca.crt pki/issued/[server-name].crt pki/private/[server-name].key /etc/openvpn/
  - 同上，生成client私钥key和证书req，以及client.crt
    - ./easyrsa gen-req [client-name] nopass
    - ./easyrsa sign-req server [client-name]
  - 创建client目录，client证书 (/etc/openvpn/client)
    - mkdir /etc/openvpn/client
    - cp pki/ca.crt pki/issued/[client-name].crt pki/private/[client-name].key /etc/openvpn/client
  - 切换到etc/openvpn，生成 tls-auth的key
    - cd etc/openvpn
    - openvpn --genkey --secret ta.key
  
### 开启ip转发
  - 编辑/etc/sysctl.conf，修改net.ipv4.ip_forward=1
    - vim /etc/sysctl.conf
  - 重新load
    - sysctl -p /etc/sysctl.conf
### 配置文件
- 模板配置文件在目录
  - ll /usr/share/doc/openvpn/examples/sample-config-files
- 配置简述
#### 修改 server 配置文件
  - cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/[server-name].conf.gz
  - gzip -d /etc/openvpn/[server-name].conf.gz
  - vim /etc/openvpn/[server-name].conf
#### 修改 client 配置文件
  - cp /usr/share/doc/openvpn/examples/sample-config-files/[client-name].conf /etc/openvpn/client
  - vim /etc/openvpn/[client-name].conf
#### 生成.ovpn客户端文件
#### 启动服务端openvpn-server
- 启动服务
  - systemctl start openvpn@[server配置文件名]
- 查看服务日志和状态
  - systemctl status openvpn@[server配置文件名]
  - journalctl -u openvpn@[server配置文件名] -xe
