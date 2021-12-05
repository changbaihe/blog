---
title: 记录Ubuntu 20.04搭建k8s集群(一) - 使用kubeadm创建集群
date: 2021-11-29 23:21:09
tags:
- k8s
categories:
- [ubuntu]
- [k8s]
---
还是老规矩，贴出官方文档 [[->(^_^)<-](https://kubernetes.io/zh/docs/home/)]


## 我与K8s的那点事
- 一直有这么个想法，自己设计产品，设计logo，开发前端，开发后端，然后流水线上线发布。

- 直到近半年，才开始一点一点上手，有尝试从研究别人的东西和构思自己的东西开始做，但是总觉得好别扭，有好多中间缺口，并且正的反馈感好弱，相对于创造性的东西，查说明书操作的事情干起来反馈感强好多，于是就调整了开始方向，先为自己搞一套服务环境吧。

- 需求来了，我需要一个小一点的服务环境，想了想用k8s吧，我是从之前公司开始接触的k8s，很幸运遇到一些很好和很厉害的人，带我get到了不少相关的东西(野路子入门)。对于野路子的我，有机会自然要亲自折腾折腾，好好体验一下，顺道野转正。

- 言归正传，下面开始对着，官方文档，开始搞，毕竟自己折腾嘛，机器有限，为了真实，我用了一台电脑主机，装成了ubuntu，准备创建一个只有一个节点(master节点)的集群试试，所有的调度都放到master上来，等之后富裕了再加work node吧。

注：以下执行的命令，都假设当前为root用户

## 准备工作 [[文档地址](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm)]
  - [检查网络适配器](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#%E6%A3%80%E6%9F%A5%E7%BD%91%E7%BB%9C%E9%80%82%E9%85%8D%E5%99%A8)
  
  - [允许 iptables 检查桥接流量](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#%E6%A3%80%E6%9F%A5%E7%BD%91%E7%BB%9C%E9%80%82%E9%85%8D%E5%99%A8)
  
  - [检查所需端口](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

## 安装 kubernetes-cni kubeadm kubelet kubectl
- 简介
  - kubernetes-cni：常见容器网络接口模块插件
  
  - kubeadm：用来初始化集群的指令
  
  - kubelet：在集群中的每个节点上用来启动 Pod 和容器等
  
  - kubectl：用来与集群通信的命令行工具

- 安装命令
  ```shell
  # 更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包
  apt-get update
  apt-get install -y apt-transport-https ca-certificates curl

  # 使用aliyun包密钥
  curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

  # 添加Kubernetes apt 仓库
  cat << EOF > /etc/apt/sources.list.d/kubernetes.list
  deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main 
  EOF

  # 安装 kubernetes-cni
  apt-get install kubernetes-cni

  # 安装 kubelet kubeadm kubectl，并锁定其版本
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl
  ```

## 容器运行时(Container Runtime) - Docker

### 官方安装说明
- [[ubuntu](https://docs.docker.com/engine/install/ubuntu/)]
- [[debian](https://docs.docker.com/engine/install/debian/)]

### 异常问题

- docker info 会显示 WARNING: No swap limit support
  - vim /etc/default/grub
    ```conf
    # 给GRUB_CMDLINE_LINUX配置添加 cgroup_enable=memory swapaccount=1 参数
    GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
    ```

  - 更新grub，重启电脑
    ```shell
    update-grub
    reboot
    ```

## kubeadm 创建集群

### 创建控制平面节点
- 前置准备
  - 关闭交换区
    ```shell
    # 关闭交换区
    swapoff -a
    # 注释掉，交换区挂载
    sed -i '/swap/s/^/#/' /etc/fstab
    ```

  - 忽略swap(不清楚为啥，总是提示不支持swap)
    ```shell
    echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/default/kubelet
    ```

- 使用kubeadm init([文档地址](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)) 创建的master节点，也可以当work节点使用，但主要功能是集群的控制面板
  - 初始化命令及参数
    
    注：kubeadm init命令的都执行了些什么 -> [[说明地址](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#%E6%A6%82%E8%A6%81)]
    ```shell
    kubeadm init \
    # k8s版本号，默认：stable-1
      --kubernetes-version [k8s版本号] \
    # 国内使用aliyun镜像源，避免卡在拉镜像上
      --image-repository registry.aliyuncs.com/google_containers \
    # 容器子网路由区间(cidr)，这个块其实可以随便写的
    # 下面cni插件我们用的是kube-flannel，它的例子配置默认是10.244.0.0/16，我们就和它保持一致
      --pod-network-cidr 10.244.0.0/16
    ```

  - 如果卡在了拉镜像上导致init慢或者失败
    
    查看kubeadm都使用了哪些基础镜像 -> [[说明地址](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-config/#cmd-config-images-list)]

    ```shell
    $ kubeadm config images list
    k8s.gcr.io/kube-apiserver:v1.22.4
    k8s.gcr.io/kube-controller-manager:v1.22.4
    k8s.gcr.io/kube-scheduler:v1.22.4
    k8s.gcr.io/kube-proxy:v1.22.4
    k8s.gcr.io/pause:3.5
    k8s.gcr.io/etcd:3.5.0-0
    k8s.gcr.io/coredns/coredns:v1.8.4
    ```
    可以使用阿里云镜像服务，搜索相应的镜像，提前docker pull下来 -> [[服务地址](https://cr.console.aliyun.com/cn-hangzhou/instances/images)]
  - 创建成功之后
    
    保存集群配置文件
    ```shell
    # 为当前用户，创建kubectl的配置读取目录
    mkdir ~/.kube

    # 拷贝管理员配置文件
    cp /etc/kubernetes/admin.conf ~/.kube/config
    ```

    保存添加work节点的参数，方便后面扩展work节点
    ```shell
    kubeadm join [ip]:[port] --token [token-val] --discovery-token-ca-cert-hash [hash-val]
    ```

- 查看kubelet日志
  - 很多问题，看kubelet错误日志，就可以很好捕获到并解决

    ```shell
    journalctl -fxeu kubelet
    ```

- 异常问题 (只要出现任何报错，执行kubeadm reset重置上次的init)
    - docker的cgroup-driver与kubelet的不一致
      
      查看各自cgroup-driver
      ```shell
      # 查看docker的cgroup-driver
      docker info | grep "Cgroup D" | cut -d" " -f4

      # 查看kubelet的cgroup-driver
      cat /var/lib/kubelet/config.yaml | grep cgroupDriver | cut -d" " -f2
      ```

      解决办法：修改docker的cgroup-driver为systemd(假设kubelet的为systemd)
      ```shell
      # 添加配置项 "exec-opts": ["native.cgroupdriver=systemd"]
      vim /etc/docker/daemon.json

      # 重启docker
      systemctl restart docker

      # 查看docker的cgroup-driver
      docker info | grep "Cgroup D" | cut -d" " -f4
      ```

### 安装cni插件(必装) - flannel

  - 这个是必装的，没有容器网络插件，coredns 就会一直pending，有很多cni，我们这里选择了flannel，见[[更多](https://kubernetes.io/zh/docs/concepts/cluster-administration/networking/#%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0-kubernetes-%E7%9A%84%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B)]
    ```shell
    mkdir ~/.kube/addons
    cd ~/.kube/addons

    # 这个需要科学上网，才能下载
    curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

    # 使用下载好的文件
    kubectl apply -f kube-flannel.yml

    # 查看flannel，运行情况
    kubectl get pods -n kube-system -o wide
    ```

  - 安装失败，最好清理cni相关目录配置文件
    ```shell
    # 这个建议慎重清理吧，主要都是kubernetes-cni安装的，可以用 dpkg -L kubernetes-cni 查看都装了些什么
    # rm -rf /opt/cni/bin/*

    # 清空cni插件相关的配置文件
    rm -rf /etc/cni/net.d/*
    ```

## END
到这里，基本上单个节点(master)的集群就创建好了，接下来(二)，我们安装【Web 界面(仪表盘)】- Dashboard