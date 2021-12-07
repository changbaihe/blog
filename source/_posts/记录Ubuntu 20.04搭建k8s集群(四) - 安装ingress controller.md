---
title: 记录Ubuntu 20.04搭建k8s集群(四) - 安装ingress controller
date: 2021-12-07 07:49:41
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

- 哎，dashboard弄好了，但是一直用api server 代理path太长有点难受，知道有ingress这么个东西，那就怎么用着怎么来吧
- 一切一切，都是为了dashboard访问舒服点，对后面的其他app部署可能一会好点，😄~

## 仔细瞅瞅文档

- 本文我们使用的是ingress-nginx，也就k8s亲生ingress controller 😄
- ingress-nginx [文档地址](https://kubernetes.github.io/ingress-nginx/)，不吹不黑，这个文档写的确实挺好，简单到位，下面简单概述下文档内容(用我这个小学生英语水平的理解)：
  - **Welcome**
    - 常见FAQ，没啥说的
    - ingress-nginx工作原理，这个可以看看，如何加载配置，热更。
    - 如何调试定位ingress-nginx的问题
    - ingress-nginx插件(这个我理解就类似于cli工具，用来操作ingress-nginx的)
  - **Deployment**：
    - Installation Guide - (安装指南)：主要介绍如何多种方式安装ingress-nginx，例helm，kubectl工具，Minikube 或者 MicroK8s工具。
      - 这块有个小坑点吧，至少我认为是，因为这块的给的例子yaml文件，是云集群的方案。。。也就是需要一个外部的Load Blancer， ingress-nginx服务type为LoadBalancer [[type类型解释详见](https://kubernetes.io/zh/docs/concepts/services-networking/service/#publishing-services-service-types)]，因为我是裸机搭的集群，没有Load Blancer。
      - 所以ingress-nginx服务类型，这块我选的是NodePort类型，直接走物理机固定端口访问ingress-nginx，这个等会
    - Bare-metal considerations - (裸机注意事项)：
      - A pure software solution MetalLB - (纯软件方案)：没太整明白咋玩的，好像是直接走的2层直接访问ingress-nginx的服务ip。
      - Over a NodePort Service - (使用NodePort映射ingress-nginx服务)：
        - 目前我选择的就是这个，对裸机来说简单，给ingress-nginx服务配置一个nodePort固定端口，通过访问node ip(也可以给node绑定外网ip使用外网ip也可)和这固定端口，来访问ingress-nginx服务。
      - Via the host network - (走host network的方式)：直接将ingress-nginx服务和宿主机用一个子网，直接访问服务的ip和端口。
      - Using a self-provisioned edge - (使用自建网络)：和云方案一样，需要一个前置的Load Blancer，就是类似完全的自建机房网络设施，我理解是暴露一个公网的代理，完事这个代理，将流量转发到整个集群，这样集群ip不会对外暴露，只对外暴露自建网络的入口代理地址。
      - External IPs - (外网ip)：给ingress-nginx服务配置外网ip，直接访问外网ip访问服务，[详见](https://kubernetes.io/zh/docs/concepts/services-networking/service/#external-ips)
    - RBAC - (基于角色的权限控制配置)：没啥好说的和前面创建dashboard的管理员角色差不多。
    - Upgrade - (如何升级已有ingress-nginx服务)
    - Hardening guide：应该是inrgess-nginx和nginx的对比吧
  - **User guide**：用户指南，这块就不一一说了。。。有点多，具体自己看吧，下面这个几个看下就能配置inrgess了。
    - ConfigMap：利用configmap配置ingress，这个我目前没去研究，不过一样的只是配置写法区别。
    - Custom NGINX template: 利用自定义模板配置ingress，这个同样暂时还没使用到。
    - Annotions：这块需要关注一下，配置ingress的时候，可以使用注解的方式进行配置就很方便，下面配置ingress使用的就这个这种方式，挺简单的。
    - TLS/HTTPS：这个是如何配置ingress-nginx使用https，例子使用的是自签证书。
  - **Examples**：好多例子，，，
  - **Developer Guide**：开发者指南，是时候给ingress-nginx贡献代码了，xdm上，我先溜了😄。

## 开始安装

- 下载yaml文件，使用裸机的deploy.yaml，见仓库[地址](https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/baremetal/deploy.yaml)

  ```shell
  # 切换目录到自定义的插件目录
  cd ~/.kube/addons
  
  # 科学上网下载，或者将下面地址使用jsdelivr转换一下
  curl -o ingress-nginx-baremetal.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
  
  # 修改一下ingress-nginx-baremetal.yaml文件，加上ingress-nginx-controller service的http和https分别一个固定的nodePort端口，例：30080，30443
  ```

- 需要修改一下ingress-nginx-baremetal.yaml，加一个固定的nodePort，默认是随机的，[端口范围详见](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#%E5%B7%A5%E4%BD%9C%E8%8A%82%E7%82%B9)

  ```yaml
  # 前面的省略这就不贴了
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
    labels:
      helm.sh/chart: ingress-nginx-4.0.10
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/version: 1.1.0
      app.kubernetes.io/managed-by: Helm
      app.kubernetes.io/component: controller
    name: ingress-nginx-controller
    namespace: ingress-nginx
  spec:
    # 使用后节点端口映射service的方式
    type: NodePort
    ipFamilyPolicy: SingleStack
    ipFamilies:
      - IPv4
    ports:
      - name: http
        port: 80
        protocol: TCP
        targetPort: http
        appProtocol: http
        # http固定的端口号
        nodePort: 30080
      - name: https
        port: 443
        protocol: TCP
        targetPort: https
        appProtocol: https
        # https固定的端口号
        nodePort: 30443
    selector:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  # 后面的省略这就不贴了
  ```

  

- 创建ingress-nginx相关工作负载，运行起来

  ```shell
  # 创建ingress-nginx
  kubectl apply -f ingress-nginx-baremetal.yaml
  
  # 查看deployment状态
  kctl -n ingress-nginx get all -o wide
  ```

## 配置ingress

#### 通过ingress配置，访问dashboard服务

- 拿前面已经安装的dashboard当例子，大概说明下当前情况和思路：

  - dashboard服务我使用的是http，就想着集群内部又不直接暴露出去，内部使用http也没啥，明文方便查问题，也少tls层这么一层(虽然内网影响不大)。
  - ingress这块对外使用https，对内访问service使用http，也就是集群内部使用http

- 下面我们开始配置：

  - 首先需要一个证书，并且将证书，托管到secret上：

    - 自签私有证书，这里就不多说了，参考dashboard文档 [详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/certificate-management.md#certificate-management)
    - 将证书托管到secret上，[详见](https://kubernetes.io/zh/docs/concepts/configuration/secret/#tls-secret)

  - 现在证书有了，我们配置ingress，[详见](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87)，这里给出一个现成的例子

    - 创建ingress.yaml，贴入如下内容：

      ```yaml
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: kubernetes-dashboard
        namespace: kubernetes-dashboard
        # 这块就是上面说的注解相关的配置，具体查文档，可以对照文档看看下面这4个注解分别啥意思
        annotations:
          kubernetes.io/ingress.class: "nginx"
          nginx.ingress.kubernetes.io/rewrite-target:  /
          nginx.ingress.kubernetes.io/ssl-redirect: "true"
          nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
      spec:
        # 这个是挂在刚才创建的secret，我这块创建的secret叫kubernetes-dashboard-ingress-secret
        tls:
        - secretName: kubernetes-dashboard-ingress-secret
        rules:
        	# 在本地配置一下host，绑定集群node ip
          - host: 'dashboard.example.com'
            http:
              paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service: 
                      name: kubernetes-dashboard
                      port: 
                        number: 80
      ```

    - kubectl apply -f ingress.yaml，这样就可以规则就生效了，打开浏览器直接访问：https://dashboard.example.com:30443 ，就可以访问dashboard了

## END

到这里，我理解k8s集群基本上就算整好了，虽然很简陋，至少，整个流程跑通了，不得不书k8s官方的概念是真的多，刚开始看着实有点头疼，不过了解了架构之后从图形流程去入手理解概念就好多了😄。接下来，去看看helm这个工具了，🤣~
