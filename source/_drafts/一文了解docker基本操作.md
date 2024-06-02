---
title: 一文了解docker基本操作
date: 2024-05-26 17:18:13
tags: docker操作
categories: Others
---

[toc]


# 摘要
本文将主要介绍docker的基本操作。日常开发过程中，偶尔会有docker的使用需求，但是经常出现忘记指令的情况，本质上还是学习的不够系统，故本文将系统地介绍大部分常用地docker基础操作。

# docker介绍与docker安装
docker可以简单看成一个轻量化的虚拟机，通过docker可以对应用程序连带环境进行快速打包与部署。
## 安装docker
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

# docker镜像
docker类比于虚拟机，那么docker镜像就可以类比于操作系统镜像，通过镜像可以还原为原来的环境和应用。

## docker pull
类似与github，docker Hub是一个存放各种docker镜像的开源仓库。通过git pull可以从docker Hub上拉取镜像
```bash
# docker [image] pull name[:TAG]
docker pull nginx
```
tag是版本号，默认情况下是nginx:latest
通过docker images可以查看本地的docker镜像
```bash
docker images
```

## docker rmi

```bash
docker rmi nginx[:TAG]
docker rmi e784f4560448 #删除image ID   
docker rmi -f nginx #强制删除
```
## docker保存与加载镜像
docker save是讲本地镜像保存到文件中，即导出到磁盘
docker load就是加载文件中的内容到镜像，即从磁盘中读取
```bash
docker save -o nginx.tar nginx:latest
docker load -i nginx.tar
```

# docker容器
docker容器可以理解为从docker镜像中实例化出来的对象。同一个镜像image可以创建多个容器container，容器之间是独立的。


# docker网络

# docker-compose编排

