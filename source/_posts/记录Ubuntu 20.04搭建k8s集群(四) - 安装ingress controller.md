---
title: 记录Ubuntu 20.04搭建k8s集群(四) - 安装ingress controller
date: 2021-12-06 17:49:41
tags:
- k8s
categories:
- [ubuntu]
- [k8s]
typora-root-url: ..
typora-copy-images-to: ..\images\k8s
---



这次先来看看这俩术语吧，这里参照nginx：

- ingress是什么，[详见](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)。
  - 基本上相当于nginx的/etc/nginx/conf.d目录中的配置文件，只是文件写法上的区别。
- ingress controller是什么，[详见](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/)。
  - 能实现按ingress规则代理请求的都属于ingress controller。
  - 其实就相当于nginx服务本身，根据ingress也就相当于配置文件，代理不同的请求。
- nginx相关的ingress controller，目前有两个官方的😄：
  - ingress-nginx：k8s官方亲生的，[仓库地址](https://github.com/kubernetes/ingress-nginx/)
  - nginx-ingress：nginx官方亲生的，[仓库地址](https://github.com/nginxinc/kubernetes-ingress)
- 俩nginx的ingress controller的区别，[详见](https://www.nginx.com/blog/guide-to-choosing-ingress-controller-part-4-nginx-ingress-controller-options/)

## 我与ingress-nginx的那点事

## 仔细瞅瞅文档

- 本文我们使用的是ingress-nginx，也就k8s亲生ingress controller 😄
- ingress-nginx [文档地址](https://kubernetes.github.io/ingress-nginx/)，不吹不黑，这个文档写的确实挺好，简单到位，下面简单概述下文档内容(用我这个小学生英语水平的理解)：
  - **Welcome**
    - 常见FAQ，没啥说的
    - ingress-nginx工作原理，这个可以看看，如何加载配置，热更。
    - 如何调试定位ingress-nginx的问题
    - ingress-nginx插件(这个我理解就类似于cli工具，用来操作ingress-nginx的)
  - **Deployment**：
    - 安装指南：主要介绍如何多种方式安装ingress-nginx，例helm，kubectl工具，Minikube 或者 MicroK8s工具。
      - 这块有个小坑点吧，至少我认为是，因为这块的给的例子yaml文件，是云集群的方案。。。也就是需要一个外部的Load Blancer， ingress-nginx服务type为LoadBalancer [[type类型解释详见](https://kubernetes.io/zh/docs/concepts/services-networking/service/#publishing-services-service-types)]，因为我是裸机搭的集群，没有Load Blancer。
      - 所以ingress-nginx服务类型，这块我选的是NodePort类型，直接走物理机固定端口访问ingress-nginx，这个等会
    - 裸机几个方案：
      - 纯软件方案，，，没太整明白咋玩的，好像是直接走的2层直接访问ingress-nginx的服务ip。
      - 使用NodePort映射ingress-nginx服务。
        - 目前我选择的就是这个，对单机裸机来说简单，给ingress-nginx服务配置一个nodePort固定端口，通过访问宿主机ip和这固定端口，来访问ingress-nginx服务
        - 如果ingress-nginx服务部署多个节点上，就需要用一个外部的Load Blancer来负载流量了更方便了。
      - 使用host network，，，好像是直接将ingress-nginx服务和宿主机用一个子网，直接访问服务的ip和端口
    - 基于角色的权限配置：没啥好说的和前面创建dashboard的管理员角色差不多
    - 如何升级已有ingress-nginx服务
    - Hardening guide：应该是inrgess-nginx和nginx的对比吧
  - User guide：用户指南，这块就不一一说了。。。有点多，具体自己看吧，只说用到的。
    - ConfigMap：利用configmap配置ingress，这个下面暂时没用到哈哈哈。
    - Custom NGINX template: 利用自定义模板配置ingress，这个下面暂时没用到哈哈哈。
    - Annotions：这块需要着重关注一下，因为配置ingress的时候，需要用的注解配置。
    - TLS/HTTPS：这个是如何配置ingress-nginx使用https，例子使用的是自签证书。
  - Examples：好多例子，，，
  - Developer Guide：开发者指南，是时候给ingress-nginx贡献代码了，xdm上，我先溜了😄。

## 开始安装

- 下载yaml文件，使用裸机的deploy.yaml，见仓库[地址](https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/baremetal/deploy.yaml)

  ```shell
  # 切换目录到自定义的插件目录
  cd ~/.kube/addons
  
  # 科学上网下载，或者将下面地址使用jsdelivr转换一下
  curl -o ingress-nginx-baremetal.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
  
  # 修改一下ingress-nginx-baremetal.yaml文件，加上ingress-nginx-controller service的http和https分别一个固定的nodePort端口，例：30080，30443
  
  # 创建ingress-nginx
  kubectl apply -f ingress-nginx-baremetal.yaml
  
  # 查看deployment状态
  kctl -n ingress-nginx get all -o wide
  ```

## END

