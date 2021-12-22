---
title: 记录Ubuntu 20.04搭建k8s集群(二) - Web界面仪表盘(Dashboard)
date: 2021-12-05 11:07:14
tags:
- k8s
categories:
- [ubuntu]
- [k8s]
---
老规矩，寻源头看一手资料，贴出相关文档地址：
- k8s官方文档 关于dashboard部分信息 [[->(^_^)<-](https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/)]
- dashboard 仓库地址[[->(^_^)<-](https://github.com/kubernetes/dashboard)]
  关于dashboard文档(仓库docs目录下)，这里我先简单介绍一下：
  - `common`：基本的文档，这个也是重点要查阅的地方。
    - dashboard容器 运行时传参args，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/common/dashboard-arguments.md)， 这个对我们来说是最有用的。
    - dashboard常见问题问答 FAQ，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/common/faq.md)。
    - dashboard规划和发展蓝图 roadmap，有兴趣可以了解下。
  - `design`：设计想法相关东西，不过多阐述。
  - `developer`：给贡献的开发者的，不过多阐述。
  - `plugins`：如何为dashboard搞插件的，不过过多阐述。
  - `user`：用户相关的安装使用，这个是重点要看一看的。
    - `installation.md`：[稳定版安装](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#official-release)/[最新开发版安装](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md#development-release)。
    - `integrations.md`：迁移相关的，不过多阐述
    - `labels.md`：dashboard仓库中标签定义描述，提issue可能需要注意，不过过多阐述。
    - `certificate-management.md`：https证书相关，如何生成自签https证书，这个有必要看看。
    - `access-control`：如何为dashboard创建管理员账号，以及绑定集群管理权限，这个必看，下面会具体说。
    - `acessing-dashboard`：如何访问dashboard，提供几种方式，这个必看，根据个人情况选择，这个下面会具体说。

## 我与dashboard的那点事
  - 单节点集群创建完了，k8s有一个ui工具dashboard，这我就不矫情了，，那肯定是要来一个，看命令行多难受，不到万不得已，谁看命令行🦆。

  - 开整开整😂~

## 开始安装

### 使用工作负载，创建dashboard
  - 我把应用的工作负载都放在，~/.kube/apps/dashboard目录下吗，方便管理，具体看个人习惯
    ```shell
    # 递归创建目录
    mkdir -p ~/.kube/apps/dashboard

    # 切换目录到dashboard
    cd ~/.kube/apps/dashboard
    ```

  - 查看文档安装说明，即上面说过的docs/user/installation.md文件，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md)，此处我选择official-release(也就是稳定发布的版本)
    
    这块需要注意一下，可能github文件下载不下来，这个时候两个选择：1.科学上网，2.免费的cdn，推荐jsdelivr转换gihtub地址，[详见转换界面](https://www.jsdelivr.com/github)。

    需要注意下，这块提供了两个选择，pod使用https还是使用http，虽然https为官方推荐，但是看个人情况吧，我的service不对外开放并且node之间也是内网环境，我选择pod使用http明文方式暴露。

    - 下载pod使用https的yaml文件, 这是官方推荐的
      ```shell
      # 科学上网的话就直接贴地址了😄
      curl -o dashboard.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

      # 使用jsdelivr转换gihtub地址
      curl -o dashboard.yaml https://cdn.jsdelivr.net/gh/kubernetes/dashboard@v2.4.0/aio/deploy/recommended.yaml
      ```

    - 下载pod使用http的yaml文件
      ```shell
      # 科学上网的话就直接贴地址了😄
      curl -o deployment.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/alternative.yaml

      # 使用jsdelivr转换gihtub地址
      curl -o deployment.yaml https://cdn.jsdelivr.net/gh/kubernetes/dashboard@v2.4.0/aio/deploy/alternative.yaml
      ```

  - 使用上面下载的yaml文件，创建dashboard，当然看个人情况可以修改yaml文件，适应自己的需求
    ```shell
    # 这里我使用的是pod使用http方式的
    kubectl apply -f deployment.yaml

    # 查看dashboard的运行情况，不出意外pod都是running，到这dashboard就创建好了
    kubectl -n kubernetes-dashboard get all

    # 若pod一直处于pending状态，执行如下操作（否则跳过），查看当前pod问题
    kubectl -n kubernetes-dashboard get pod [podname] -o go-template --template={{.Events}} && echo
    ```

  - pod一直pending常见原因：
    
    - 单节点集群：k8s默认master节点被标记taint，pod一般tolerations不容忍这个taint，导致pod不会被调度到master节点上，并且一直处于等待被调度中，有如下两种解决办法如下：
      - 取消master的taint标记，让master参与被调度。

        ```shell
        # 命令详情：kubectl taint -h
        kubectl taint nodes [nodename] node-role.kubernetes.io/master-
        ```
      - 新加一个work node到集群中。



### 创建admin账号(ServiceAccount)，文档即上面说的docs/user/access-control目录下，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)
  - 上面下载的yaml文件中，默认的ServiceAccount权限太小，只能访问namespace为default的
    ```shell
    # 创建dashboard-adminuser.yaml文件
    echo << EOF > dashboard-adminuser.yaml

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard

    ---

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    
    EOF

    # 创建admin-user用户
    kubectl apply -f dashboard-adminuser.yaml

    # 查看admin-user
    kubectl describe sa admin-user -n kubernetes-dashboard
    ```
  
  - 删除用户及校色绑定关，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md#clean-up-and-next-steps)

  - 获取adminuser用户token，注意保存token，用户登录dashboard

    ```shell
    kubectl \
    -n kubernetes-dashboard \
    get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") \
    -o go-template="{{.data.token | base64decode}}" \
    && echo
    ```

### 访问dashboard，文档即docs/user/accessing-dashboard目录下，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md)

##### 总共给出了4种方式访问dashboard

  - 本地访问，临时代理，只能localhost/127.0.0.1访问，我理解是本地调试用的
    - kubectl proxy访问，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md#kubectl-proxy)

    - kubectl port-forward访问，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md#kubectl-port-forward)

  - NodePort直接暴露service到宿主机指定端口
    - 相当于直接把pod暴露出去了，不是最好的方式，上面我选择的pod使用http当时暴露，这种方式我就跳过了。

    - 如果需要使用过这种方式，修改一下上面的deployment.yaml，将service的`type: ClusterIP`换成`type: Node Port`，然后为service指定一个固定的nodePort端口(不指定就是随机)，[详见](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md#nodeport)

  - Ingress代理，这个后面安装ingress的时候，用dashboard当作例子专门说下如何配置。

  - API Server代理访问，这个path太长。。。。

##### 暂时选择API Server代理，来作为本篇访问dashboard的方式，下面是具体说明：
  - 由于我们的api server使用的是 默认的自签证书的https，我们需要导出一份证书给浏览器用，否则浏览器访问403

    ```shell
    # kubeadm init 默认k8s配置目录：/etc/kubernetes/
    # 我们创建一个client目录，用于导出证书
    mkdir /etc/kubernetes/client && cd /etc/kubernetes/client

    # 创建证书
    grep 'client-certificate-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

    # 创建key
    grep 'client-key-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

    # 生成kubecfg.p12证书文件
    openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -name "kubernetes-client" -out kubecfg.p12
    ```

  - 将生成的证书文件 kubecfg.p12 导入到浏览器中，假设使用chrome浏览器，具体步骤如下:
    1. 点击右上角三个点，选择设置

    2. 点击 隐私设置和安全性，选择安全
    
    3. 然后选择 管理证书，这时会弹出一个非浏览器界面
    
    4. 点击 导入，按照默认步骤导入即可
  
  - 浏览器访问dashboard，kubeadm init创建集群时若没有指定api server端口，默认端口是6443
    - https://[ip地址]:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

    - 这块有两种验证方式

      - kubeconfig方式，这里就不多说了，看文档整一份kubeconfig配置文件，[详见](https://kubernetes.io/zh/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

      - token方式，将上面admin-user的token填入即可

## 遗留问题
  - dashboard-metrics-scraper请求api server的metrics api调用异常，并且dashboard界面不显示CPU Usage和Memory Usage
    - 上篇创建集群，没有安装metrics-server，所有这块dashboard是没办法通过k8s的metrics api采集到node和pod的信息的，dashboard上就不会显示CPU Usage和Memory Usage图表。
    
    - 不过这不影响dashboard的正常使用，下篇安装了metrics-server，CPU Usage和Memory Usage就有了。

## END
到这dashboard就整好了，集群的所有状态和配置，尽收眼底，使用界面管理，舒服了~