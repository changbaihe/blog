---
title: 记录Ubuntu 20.04搭建OpenVPN服务
date: 2021-11-23 22:30:42
tags:
- openvpn
categories:
- ubuntu
---
依旧是那个位置，依旧是那个笑脸，[[->(^_^)<-](https://ubuntu.com/server/docs/service-openvpn)] (ubuntu-server版openvpn原文档)，没错，我是“docs党”，同时也是“google党”和“baidu党”，哈哈哈~

## 我与OpenVPN的那点事
- 节奏1：mac上用Tunnelblick，，，，简直恶心，，，版本问题好多，被恶心到好几次，然后对配置也不了解，有问题了也不知道咋解决，化身google党，baidu党，稀里糊涂凑合用，之后发现Viscosity这款软件不错，就很少出问题，虽然配置仍然不懂。。。这个时候已经有想去好好研究下的想法了。
- 节奏2：就在前不久，我赶着win11热乎发布，我重装凑合用了一年多的win10(之前觉得系统有点乱，也没找到一个合适点去重装系统，也是懒的重复折腾系统那点事)，确实凑合久了，装系统的时候很痛快的，直接一把all in —— 格式化C盘，事后，就顿时酸爽了，我的vpn啥的忘备份出来了。。。那就从新装吧，作为一个docs党嘛，自然是去找它的官网了，下载最新客户端(就是喜欢体验新版本)，，，，下载下来，使用原来的证书和配置似乎连不上。。。~[此处略过一段悲伤的记忆]~，记得当时证书一直报错啥tap啥的，看配置一时间又整不明白，毫无头绪。。。最后下载旧版2.3尝试，最后算是可以了。。。这个时候心里对它开始躁动了~
- 节奏3：时间来到了最近，整了一个二手的tower主机，我装了一个ubuntu-server镜像，放在屋里当服务器学习，一开始，路由器上配置了自己电脑的远程桌面的端口，想着在公司可以远程自己的电脑，省的带电脑。碰巧就遇到一个ssh的问题，想在自己服务器上试试ssh相关的操作。于是，我通过远程面屋里的电脑，登录到路由器配置页，开始配置端口，这个时候脑残的一幕出现了，手一哆嗦点到了删除所有端口映射。。。。当场歇菜。。。顿时蒙蔽。。。眼前千万个草泥马奔腾而过~~~又想笑又无奈，只能等晚上回去搞了。这个时候OpenVPN慢慢的在我的脑海中升起来了，，，就好像说，来吧是时候展现真正的技术了，把我搞了，这不随便整，配一次端口就行了，还要啥自行车呢。嗯，有道理，那就搞喽(我忍你很久了)~
- 行动：干它鸭的，开搞开搞~

## 搭建过程 (看着↑头提到的文档)

### 安装工具包和证书生成 (假设当前是root用户)

- 安装openvpn 和证书工具easy-rsa
  ```shell
  apt update
  apt install easy-rsa
  # 默认目录为/etc/openvpn
  apt install openvpn
  ```
- 备份easy-rsa脚本 (软链接了一下当前全局的脚本，防止更新，差不多算锁版本吧)
  ```shell
  $ make-cadir /etc/openvpn/easy-rsa
  ```
- 初始化pki目录和相关模板和创建公共ca.cert以及dh算法参数文件
  ```shell
  # 切换到easy-rsa目录
  cd /etc/openvpn/easy-rsa

  # 初始化pki目录  => pki/*
  ./easyrsa init-pki

  # 创建公共ca.cert => pki/ca.crt
  ./easyrsa build-ca

  # 生成dh算法参数 => pki/dh.pem
  ./easyrsa gen-dh

  # 生成双向验证的tls-auth的key
  openvpn --genkey --secret ta.key
  ```
- 生成server证书私钥，[server-name]替换成自定义name，并移至根目录/etc/openvpn
  ```shell
  # 生成证书req文件
  ./easyrsa gen-req [server-name] nopass

  # 使用./vars文件附带参数 和 证书请求，生成证书
  ./easyrsa sign-req server [server-name]

  # cp 证书到根目录
  cp ta.key pki/dh.pem pki/ca.crt pki/issued/[server-name].crt pki/private/[server-name].key /etc/openvpn/
  ```
- 生成client证书和私钥，[client-name]替换成自定义name
  ```shell
  # 生成证书req文件
  ./easyrsa gen-req [client-name] nopass

  # 使用./vars文件附带参数 和 证书请求，生成证书
  ./easyrsa sign-req server [client-name]

  # 创建client文件夹
  mkdir /etc/openvpn/client

  # cp client的所需文件到 /etc/openvpn/client
  cp pki/ca.crt pki/issued/[client-name].crt pki/private/[client-name].key /etc/openvpn/client
  ```

### 开启ip转发

- vim /etc/sysctl.conf, 确保net.ipv4.ip_forward=1
  ```conf
  # 这个默认好像是被注释掉的，解开注释就行
  net.ipv4.ip_forward=1
  ```
- 重新load
  ```shell
  sysctl -p /etc/sysctl.conf
  ```

### 配置文件

#### Tips：

- client/server配置问题 [[文档地址](https://openvpn.net/community-resources/how-to/)]
- openvpn 命令详情 [[文档地址](https://community.openvpn.net/openvpn/wiki/Openvpn24ManPage)]
- 模板配置文件在目录下
  ```shell
  ll /usr/share/doc/openvpn/examples/sample-config-files
  ```

#### 修改 server 配置文件
- 复制server模板文件
  ```shell
  # 复制模板配置文件压缩包
  cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/[server-name].conf.gz

  # 解压
  gzip -d /etc/openvpn/[server-name].conf.gz
  ```
- vim /etc/openvpn/[server-name].conf
  ```conf
  # 当前监听的外网ip，可以不指定
  ;local a.b.c.d
  # server监听的端口号
  port 1194
  
  # 当前服务采用啥协议，tcp和udp两种
  proto tcp
  ;proto udp

  # 路由模式，vpn的子网ip pool，10.8.0.0/24 最多支持同时256个客户端连接
  server 10.8.0.0 255.255.255.0

  # 网桥模式，闲麻烦就用路由模式
  ;server-bridge
  ;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

  # 使用设备类型：tap => 网桥模式， tun => 路由模式
  dev tun
  ;dev tap

  # 给客户端推送添加，指定私有子网路由，如果需要通过vpn server访问内部子网，在这配置一下响应的子网路由
  push "route 192.168.1.0 255.255.255.0"
  ;push "route 172.24.0.0 255.255.255.0"

  # server 状态日志文件路径地址
  status openvpn-status.log
  # server 日志文件路径地址
  log-append  openvpn.log
  # server 日志级别
  verb 3
  # 连续至少20条重复的消息日志，不打印到log，根据需要开关
  ;mute 20

  # 下面这几个参数，猜测和https的tls握手过程获取的内容是一样的，vpn只不过直接都拿到本地了，不再需要tls去获取了
  # 公共证书
  ca ca.crt
  # tls加密层私钥key，server => 0，client => 1
  tls-auth ta.key 0
  # dh加密算法参数文件
  dh dh.pem
  # 加密密码
  cipher AES-256-CBC

  # server公开证书
  cert myservername.crt
  # server的私钥key
  key myservername.key

  # ping频率(秒/次) alive心跳最长周期
  keepalive 10 120

  # k-v缓存，client的ip，断线重连，分配相同ip，根据需要开关
  ifconfig-pool-persist /var/log/openvpn/ipp.txt
  # 可以让客户端相互ping通访问
  client-to-client
  # 关闭则一个证书只能一个client使用连接，打开则可一个证书多端连接
  duplicate-cn

  # 客户端断线重连次数，只能udp模式下使用，否则tcp模式server启动报错
  ;explicit-exit-notify 1

  # 客户端并发连接数
  ;max-clients 100

  # 这俩暂时没搞清楚是怎么个作用
  persist-key
  persist-tun

  # server 压缩方式
  ;compress lz4-v2
  # 推送给client的压缩方式
  ;push "compress lz4-v2"

  # 兼容旧版本client的压缩方式，需要客户端也配置这项
  ;comp-lzo

  # 给公共的client证书，绑定一个子网，供这个证书的client连接分配
  ;client-config-dir ccd
  ;route 192.168.40.128 255.255.255.248

  # 通过脚本来控制不同client的用户/用户组，
  ;learn-address ./script
  # 配置client，通过vpn，重定向到默认网关
  ;push "redirect-gateway def1 bypass-dhcp"
  # 配置client dns
  ;push "dhcp-option DNS 208.67.222.222"

  # 下面这个几个参数具体没整明白是怎么个作用
  # 拓扑子网啥的
  ;topology subnet
  # wins下的新网桥
  ;dev-node MyTap
  # 非wins下可以开启，好像是liunx系统下的，共享用户和用户组
  ;user nobody
  ;group nogroup
  ```

#### 修改 client 配置文件
- 复制client模板文件
  ```shell
  # 复制模板配置文件
  cp /usr/share/doc/openvpn/examples/sample-config-files/[client-name].conf /etc/openvpn/client
  ```
- vim /etc/openvpn/client/[client-name].conf
  ```conf
  client
  
  # vpn server地址
  remote [ip] [port]
  # 通过tls层，进行server证书验证
  remote-cert-tls server
  # 解析域名重试次数，无限次或者指定次数
  resolv-retry infinite
  # 不绑定本地固定端口号
  nobind

  # 走http代理模式连接vpn
  ;http-proxy [proxy server] [proxy port ]
  ;http-proxy-retry

  # 同server一样, ，tcp和udp两种
  proto tcp
  ;proto udp
  # 同server一样, 设备类型：tap => 网桥模式， tun => 路由模式
  dev tun
  ;dev tap

  # 同server一样
  ca ca.crt
  # 同server一样, tls加密层私钥key，server => 0，client => 1
  tls-auth ta.key 1
  # 同server一样, 加密密码
  cipher AES-256-CBC

  # client公开证书
  cert client.crt
  # client私有key
  key client.key

  # 这俩暂时没搞清楚是怎么个作用
  persist-key
  persist-tun

  # server 日志级别
  verb 3
  # 连续至少20条重复的消息日志，不打印到log，根据需要开关
  ;mute 20
  # wifi模式下，会产生很多重复包，这个开关时静音，重复警报
  ;mute-replay-warnings

  # 旧版本client的压缩方式，服务端若开启，则客户端也需要配置
  ;comp-lzo

  # 非wins下可以开启，好像是liunx系统下的，共享用户和用户组
  ;user nobody
  ;group nogroup
  ```

#### 生成.ovpn客户端文件
- conf和.ovpn的区别，就是conf文件，加4个标签数据
  ```conf
  # 同conf内容
  <ca>……</ca>
  <cert>……</cert>
  <key>……</key>
  <tls-auth>……</tls-auth>
  ```
- 切换目录到/etc/openvpn/client
- vim makeovpn.sh
  ```sh
  echo "Please enter an existing client.conf Name:"
  read Name

  OvpnFile=$Name.ovpn

  cat $Name.conf > $OvpnFile
  echo "<ca>" >> $OvpnFile
  cat ca.crt | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' >> $OvpnFile
  echo "</ca>" >> $OvpnFile

  echo "<cert>" >> $OvpnFile
  cat $Name.crt | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' >> $OvpnFile
  echo "</cert>" >> $OvpnFile

  echo "<key>" >> $OvpnFile
  cat $Name.key >> $OvpnFile
  echo "</key>" >> $OvpnFile

  echo "<tls-auth>" >> $OvpnFile
  cat ta.key >> $OvpnFile
  echo "</tls-auth>" >> $OvpnFile

  echo "Done! $OvpnFile Successfully Created."
  ```
- ./makeovpn.sh

#### 私有内网路由转发配置
- 假设openvpn server端口号时1194，子网：10.8.0.0/24
- 防火墙配置 (这里需要注意, ubuntu 有自己的ufw，我们选择使用iptables配置，就关掉ufw，否则可能有影响)

  ```shell
  # 禁用防火墙
  ufw disable
  
  # 配置规则
  iptables -I INPUT -p tcp --dport 1194 -m comment --comment "openvpn" -j ACCEPT
  iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE
  iptables-save > /etc/sysconfig/iptalbes
  ```

#### 启动服务端openvpn-server
- 启动服务
  ```shell
  # 启动openvpn 服务
  systemctl start openvpn@[server配置文件名]
  ```
- 查看服务日志和状态 (出问题了，可以把server日志级别调到最大9，方便排查问题)

  ```shell
  # 查看openvpn 服务状态
  systemctl status openvpn@[server配置文件名]

  # 查看日志
  journalctl -u openvpn@[server配置文件名] -xe
  ```

#### 客户端启动
- wins
  - OpenVPN[官网下载](https://openvpn.net/community-downloads/)client安装包，将上面生成.ovpn文件导入即可
- mac
  - Tunnelblick用的是真的恶心，这里推荐，使用【Viscosity】，真的稳定好用
  - 操作步骤:
    - 将.ovpn文件导入Viscosity
    - 编辑导入的配置，选中[验证]菜单，将选项Direction，设置成1，这个对应与上面的tls-auth ta.key 1
    - 保存，连接即可
- liunx (暂时没这个需求，后续整理完补上吧)

## END
到这openvpn server就搞好了，我们可以愉快的，随时通过vpn，远程自己家里的电脑了🤭。目前client直接走的私有证书连接的，没有配置账号，有需要走账号密码的，问题不大，根据配置文件自己调整。
