---
title: 记录Ubuntu 20.04搭建k8s集群(三) - 安装metrics-server
date: 2021-12-06 11:50:34
tags:
- k8s
categories:
- [ubuntu]
- [k8s]
typora-root-url: ..
typora-copy-images-to: ..\images
---

 这次变了😄，文档下面详说下，强调一下当前背景：

- 这是基于之前的搭建情况下进行的，条件有限，我们当前使用的是单节点集群(只有一个master，既当控制面又当工作节点)。

## 我与Metrics Server的那点事

- 本来是不想先整metrics的，dashboard弄好了，本来挺舒服的，又去dashboard仓库转了一圈，看到了下面这张图，然后就发现，我的CPU Usage 和Memory Usage呢？？？

  ![](/images/dashboard-sample.jpg)

- 然后Google一下，，，有人说是Metrics API有问题，就去看了一下，dashboard-metrics-scraper日志，果然。。。k8s自己本身api server 不提供Metrics数据指标整合接口实现。。。kubectl top 命令也是显示` error: metrics not available yet`。。

- 没办法，那整一下Metrics Server吧😔

## 仔细分析一波文档

- k8s官方文档中Metrics API 相关介绍，[详见](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/resource-metrics-pipeline/)

  - API Server没有提供Metrics API的具体实现，k8s只提供了一套API定义供相关第三方工具去自己实现相关指标整合，并将Metrics接口服务注册到API Server。

  - 实现Metrics API的指标采集工具，目前我了解到的是下面这俩：
    - Prometheus：这个之前有使用过，搭配Alertmanager和Grafana，做服务预警和指标监控，就感觉挺爽。

    - Metrics Server：这个看文档说的好像是没有Prometheus专业，也不推荐Prometheus直接使用它的指标数据，因为它的数据是实时的没有提供持久化(这里我就理解它为轻量一点吧😄)。

  - 为了让dashboard能采集到node和pod的相关指标，以及kubectl top可以使用，就怎么简单怎么来，选择Metrics server作为Metrics API实现server。

- Metrics Server仓库地址 [[->戳这<-](https://github.com/kubernetes-sigs/metrics-server)]，下面我们来简述一下，对我们有用的文档内容：

  - 安装前置条件

    - Metrics Server到API server的网络是可以访问的。

    - k8s API Server是开启聚合层配置的，[详见](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/)。

      - 这块主要是双向鉴权相关的内容(即从client request => api server => metrics server之间的双向鉴权)。

    - 需要开启Kubelet 认证/鉴权，[详见](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)，这个应该创建集群的时候，不特殊指定，默认开启的：

      - 这块主要是 metrics server => kubelet 提供的指标采集api 之间的鉴权。

      - kubelet默认工作目录：/var/lib/kubelet

      - 查看配置文件/var/lib/kubelet/config.yaml，可见webhook是开启的，并且使用的x509认证，证书就是集群自签的ca.crt

        ```shell
        # 有点长，这里之查看前15行内容
        head --line 15 config.yaml
        apiVersion: kubelet.config.k8s.io/v1beta1
        authentication:
          anonymous:
            enabled: false
          webhook:
            cacheTTL: 0s
            enabled: true
          x509:
            clientCAFile: /etc/kubernetes/pki/ca.crt
        authorization:
          mode: Webhook
          webhook:
            cacheAuthorizedTTL: 0s
            cacheUnauthorizedTTL: 0s
        cgroupDriver: systemd
        ```

    - kubelet使用的ca需要是集群的ca，这个确实是集群的ca，上面的clientCAFile: /etc/kubernetes/pki/ca.crt就是集群的ca，
      - 自签的这个证书上并没有绑定具体的node ip，就会产生x509校验不过问题。
      - 最简单的办法，在启动Metrics Server容器时使用--kubelet-insecure-tls，使用不安全的tls访问kubelet的指标接口。
      - 这个x509校验要是不过，Metrics Server是没办法注册到API Server的。
    - 容器运行时，需要实现[Container Metrics RPCs](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/cri-container-stats.md) ，我们使用的是Docker，这个具体没有去研究，也是有一套API，Docker肯定是实现了，不然kubelet也不可能采集到Pod的数据
      - 这个主要是kubelet去采集当前node上的各个pod的指标数据

  - 容器启动参数

    - --kubelet-preferred-address-types：访问节点地址，优先使用顺序，默认：[Hostname, InternalDNS, InternalIP, ExternalDNS, ExternalIP]
    - --kubelet-insecure-tls：这个就是上面说的，可以不校验ca，文档上说仅对测试目的这么用，但是自己学习环境，也差不多。
    - --requestheader-client-ca-file：指定一个根证书文件，用于发起请求的客户端(也就是Metrics Server)的验证。

  - 提供helm安装和yaml文件直接安装，下面我们使用yaml文件安装

##  开始安装

- 下载yaml文件, 这块文档提供的github地址不是最终文件地址，需要重定向跳转，没办法使用jsdelivr转换地址，只能科学上了。

  ```shell
  # 切换目录到自定义的插件目录
  cd ~/.kube/addons
  
  # 科学上网下载
  curl -o metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -L
  
  # 修改一下metrics-server.yaml文件加上--kubelet-insecure-tls参数
  
  # 创建metrics-server相关负载
  kubectl apply -f metrics-server.yaml
  
  # 查看deployment状态
  kctl -n kube-system get deployment.apps/metrics-server
  ```

## END

Metrics Server安装完了，再去dashboard上看看，CPU usage 和 Memory usage有了，还有需要注意得一个地方，访问dashboard的/overview路径是对应的上面的那个例图，我的Dashboard v2.4.0+0.ge75ebcf68，点击logo跳转的是/workloads，不是/overview。。。。反正是访问/overview就和上面那个例图一样了。。😄

