---
title: 命令行连接mongo.md
date: 2021-11-18 18:11:47
tags:
  - mongodb
categories:
  - dbs
---

依旧是话不多说，看一手资料，献祭出，mongodb官网文档：[[->(*^_^*)<-](https://docs.mongodb.com/mongodb-shell/install/)]，工欲善其事，必先看文档，对官方文档很了解了，遇到啥问题，也不慌。

## 我与mongosh的无奈之缘

- 但凡能用ui界面工具，谁又会去想着用命令行zx呢，哎都是被逼的，就在月前，运维通知，禁止内网直接连接线上mongo，这一波操作，对于习惯用ui工具的简直恶梦
- 那么只有用命令行了，印象中之前使用命令行都是装mongo自带的，就只想装一个连接mongo的工具，装啥呢，赶紧去扒拉官网文档看有没有现成的，真幸运，官网刚好提供连接工具mongosh，并且支支持4.0及以上版本，更幸运的是，我们数据库是4.0, 顿时又特么欢快起来了，开搞开搞，装它丫的

## 安装mongosh (假定当前为root用户)
  - 选操作系统，我用的是Ubuntu 20.04 (Focal)
    - lsb_release -dc
  - 安装包公共key(GPG)
    - apt-get update && apt-get install gnupg
    - wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
  - 创建mongo源list文件，并写入源地址
    - echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | tee /etc/apt/sources.list.d/mongodb.list
  - 安装 mongosh
    - apt-get update
    - apt-get install -y mongodb-mongosh
  - 查看 mongosh 本版
    - mongosh --version

## 连接到mongo
  - 参数选项 [[文档地址](https://docs.mongodb.com/mongodb-shell/reference/options/)]
  - 常见使用方式
    - 连接单点模式
      - mongosh mongodb://[host]:[port]/[db] --authenticationDatabase [权限db] -u [用户名] -p [密码]
    - 连接集群模式-复制集
      - mongosh mongodb://[host1]:[port1],[host2]:[port2],[host3]:[port3]/[db]?replicaSet=[复制集] --authenticationDatabase [权限db] -u [用户名] -p [密码]

## mongosh当前支持的方法api [[文档地址](https://docs.mongodb.com/mongodb-shell/reference/methods/)]
  - 查看数据库版本号
    - db.version()
  - 查看当结点是否是master
    - db.isMaster()
  - 查看当前mongo实例信息
    - db.hostInfo()