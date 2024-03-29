---
layout: post
title: docker 的四种网络模式
categories: 容器网络
description: docker 网络基础 
keywords: docker network 
---
docker 的基础网络模型，描述了容器与外部通信的不同线路，涉及对象包括容器和宿主机。  

#### 1、none
如果容器不使用网络，那么容器与其他容器，容器与宿主机都没有任何关系，这种网络模式为 none。
![docker_network_none](http://wyong.cn/images/blog/docker/docker_network_none.png)
#### 2、container
如果容器使用网络，但是容器只与其他容器有关系，共用其他容器网络栈，容器与宿主机没有直接关系，这种网络模式为 container。
![docker_network_container](http://wyong.cn/images/blog/docker/docker_network_container.png)
#### 3、host
如果容器使用网络，但是容器只与其他容器没有有关系，容器直接借助宿主机的网络栈对外提供服务，这种网络模式为 host。
![docker_network_host](http://wyong.cn/images/blog/docker/docker_network_host.png)
#### 4、bridge
如果在宿主上创建一个虚拟交换机，将宿主机上所有的容器连接到此交换机，这种网络模式为桥接。
![docker_network_bridge](http://wyong.cn/images/blog/docker/docker_network_bridge.png)