---
title: Docker
date: 2022-6-25 10:02:51
tags: 学习笔记
categories: 学习笔记
---

### 介绍

Docker是一个开源的应用容器引擎，它允许开发者将应用及其依赖项打包到一个可移植的容器中，然后发布到任何安装了Docker引擎的服务器上。 Docker的核心思想是容器化，通过将应用程序及其依赖项打包成一个容器，使得应用程序在不同的环境中都能够快速可靠地部署和运行。这种技术可以大大简化应用程序的部署和管理，提高应用程序的可移植性和安全性。

Docker的主要特点包括：

- 轻量级虚拟化：Docker基于轻量级虚拟化技术，通过容器化实现应用程序的运行环境隔离，而不需要模拟整个操作系统。这使得Docker容器启动速度快、资源消耗低。
- 跨平台兼容性：Docker可以跨平台安装和使用，支持在Linux、Unix系统以及Windows等操作系统上运行。
  镜像机制：Docker使用镜像来创建和运行容器。镜像是一个只读的模板，包含了运行应用程序所需的所有文件系统、库和依赖项。通过使用Docker镜像，开发者可以快速、可靠地构建和部署应用程序。
- 容器隔离：Docker容器之间相互隔离，每个容器都有自己的文件系统、进程空间和网络接口。这种隔离性使得容器可以在同一台主机上同时运行多个应用程序，而不会相互干扰。
- 仓库管理：Docker仓库用来存储和管理Docker镜像。仓库可以分为公共仓库和私有仓库两种类型，其中公共仓库如Docker Hub提供了大量的官方和社区维护的镜像供用户下载和使用。

通过使用Docker，开发者可以更加方便地构建、部署和管理应用程序，提高开发和运维效率，实现应用程序的快速交付和可移植性。

### 常用命令

1.启动docker服务

```undefined
 sudo service docker start
```

2.停止docker服务

```undefined
 sudo service docker stop
```

3.检查docker 守护进程是否在运行

```undefined
 sudo docker stats
```

4.查看docker相关信息

```undefined
 sudo docker info
```

5.列出所有容器

```undefined
sudo docker ps -a
```

6.最后一次运行的容器

```undefined
sudo docker ps -l
```

7.重新启动已停止的容器

```undefined
sudo docker start 容器名(也可以使用容器ID)
```

8.获取容器的日志

```undefined
sudo docker logs 容器名
```

获取最后几条日志

```undefined
sudo docker -f 容器名
```

9.列出镜像

```undefined
sudo docker images
```

10.拉取镜像

```undefined
sudo docker pull 镜像名
```

11.删除所有容器

```jsx
sudo  docker rm $(docker ps -a -q)
```

12.删除单个容器

```undefined
sudo docker rm 容器名
```

13.删除所有镜像

```ruby
sudo docker rmi $(docker images | grep none | awk '{print $3}' | sort -r)
```

14.保存镜像

```jsx
sudo docker save 镜像名 > /home/新镜像名.tar
```

14.加载自定义镜像

```jsx
sudo docker load < /home/自定义镜像
```

15.获取容器更多信息

```undefined
sudo docker inspect 容器名
```

16.删除为none的镜像

```dart
docker images --no-trunc| grep none | awk '{print $3}' | xargs -r docker rmi
```

### 常用参数：

> -i：以交互模式运行容器，通常与 -t 同时使用
>  -t：为容器重新分配一个伪输入终端，通常与 -i 同时使用
>  -p : 端口映射 格式为[主机端口：容器端口]
>  -d : 后台模式运行
>  -name : 给容器一个新的名称
>  -v：挂载主机的目录
>  -e： username="ritchie": 设置环境变量
>  -m：设置容器使用内存最大值
>  --env-file=[]：从指定文件读入环境变量