---
layout: post
title: k8s 应用故障排查 
categories: kubernetes
description: k8s 应用故障排查
keywords: k8s kubectl-debug livenessProbe readinessProbe telepresence 
---
当应用放进容器，就给应用故障排查带来了新的挑战，因为容器环境的隔离，以及调试命令的缺失。但是在排查问题的时候，我需要能像操作虚拟机一样操作容器，还好 k8s 社区庞大，有足够的工具可以排查 k8s 应用。

## 一、 容器应用探测
如果应用没能成功运行起来，一般都是什么问题呢？当应用运行起来以后，怎么来保证应用是正常运行的呢？怎么确保应用一直处于正常对外提供服务中？在 k8s 中，有三种应用探测方式，tcp、http、command。

#### 1、pod 的常见状态及原因
* Pending，pod 等待调度，调度器没有找到合适的 node，可能的原因有
    - 调度器没有给 pod 绑定 node
    - 通常原因 cpu、memory、nodeselector等
* waiting，pod 还没有进入调度状态，在等待调度，可能的原因有
    - 镜像，私有仓库、镜像不存存在等
* crashing，因为容器内应用启动失败，pod 不断的重启，可能的原因有
    - 应用配置等有问题，导致不断的拉取镜像启动容器，但是容器启动失败
* Runing，应用运行正常
    - 处于 Running 状态的 pod 可以正常对外提供服务
    - 如果没能对外提供服务，可能是 service label 匹配问题，service 和 pod 端口映射等

#### 2、探针使用
探针基础
![k8s-livenessProbe-readinessProbe](http://wyong.cn/images/blog/k8s/livenessProbe_readinessProbe.png)
网络探针存在于 kubelet ProbeManager 中，是其代码实现检查。
##### I、tcp 案例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: tcp-probe-test
  name: tcp-probe-test
spec:
  replicas: 1
  selector:
    matchLabels:
      run: tcp-probe-test
  template:
    metadata:
      labels:
        run: tcp-probe-test
    spec:
      containers:
      - image: nginx:alpine
        name: tcp-probe-test
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 15
          timeoutSeconds: 1
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 1
          periodSeconds: 3
```
##### II、http 案例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: http-probe-test
  name: http-probe-test
spec:
  replicas: 1
  selector:
    matchLabels:
      run: http-probe-test
  template:
    metadata:
      labels:
        run: http-probe-test
    spec:
      containers:
      - image: nginx:alpine
        name: http-probe-test
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          timeoutSeconds: 1
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          timeoutSeconds: 1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```
##### III、command 案例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: command-probe-test
  name: command-probe-test
spec:
  replicas: 2
  selector:
    matchLabels:
      run: command-probe-test
  template:
    metadata:
      labels:
        run: command-probe-test
    spec:
      containers:
      - image: myimage:1.0
        name: deploy-graceful
        livenessProbe:
          exec:
              command: ["echo","livenessProbe"]
        readinessProbe:
          exec:
              command: ["echo","readinessProbe"]
      terminationGracePeriodSeconds: 3
# command 使用注意：
# 第一个命令后面的，都是第一个命令的参数
```
    

## 二、 容器应用故障排查
容器一般为了保证体积比较，一般在容器内部找不到调试命令，kubectl-debug 工具就出现了。  
在排查网络的时候，一般很容易请求容器内部的服务，但有时需要让集群内部的应用调用本地服务，这时 telepresence 这个反向链路调试工具就出现了。

#### 1、通过日志排查
```
kubectl logs -f POD_NAME
```
当 pod 中有两个及以上容器的时候，需要使用 -c 加上容器名字
```
kubectl logs -f alertmanager-prometheus-operator-alertmanager-0 -c alertmanager
```
**注意，需要确定错误日志都是通过标准输出输出出来的，如果有输出到文件的还需要登陆容器仔细排查**

#### 2、正向链路排查
```
kubectl port-forward service/SERVICE_NAME local_PORT:service_PORT
```
#### 3、反向链路排查
**通过 telepresence 让集群内部的应用访问到本地的服务**
这里没有仔细查看 telepresence 文档，只看了一个简单的用法，然后稍加修改，达到了想要的结果。  
操作步骤

* 创建一个 deployment，并使用 service 暴漏端口
```
kubectl run test --image="nginx:alpine"
kubectl expose deploy test --port=80 --target=80
```
* 缩容 deployment pod 数量到 0
```
kubectl scale deploy test --replicas=0
```
* 执行 telepresence 命令将本地服务代理到集群内部
```
telepresence --swap-deployment test --expose 80:80
```
* 集群内部通过 原 service 就可以访问到本地服务  
启动本地应用监听在 80 端口

#### 4、 使用 kubectl-debug 排查
kubectl-debug 的原理是，启动定义镜像，将其加入到需要调试的 pod 的 namespace 中。
kubectl-debug 使用案例
```
k debug  sample-app-656b98c5d8-4zcqw --agentless=true --port-forward=true --agent-image=wvalianty/debug-agent:v0.1.1
```
如果 pod 的状态为 crashing，需要加``` --fork ```参数，具体排查类似在虚拟机中的排查。

## 三、 k8s 集群节点维护
在集群运行过程中，可能需要维护某一个节点，尤其是自建集群的时候，维护 node 节点是必不可少的操作。但是 node 上面可能有正在运行的服务，那么一般怎么操作呢？

#### 1、如果 node 上面没有 pod，直接标记 node 不可调度即可
```
k cordon NODE01
```
维护结束，直接通过下面命令恢复调度
```
k uncordon NODE01
```
#### 2、 如果 node 上有正在运行的 pod
如果 node 上有正在运行的 pod，则需要考虑 pod 重启对业务的影响，尤其该业务只有一个 pod 在该 node 上时。如果确保业务没有问题的话，可以通过 drain 将该节点的 pod 驱逐掉。
```
k drain NODE01 --ignore-daemonsets --delete-local-data
```
维护结束，通过```uncordon```重新恢复调度。
#### 3、 节点故障排查经验
公司一套 redis 集群，运行于 k8s 集群上，发现当 redis pod 调度到某一台服务器的时候，连接会有些问题。  

当使用 redis 客户端命令进行连接的时候，还可以正常建立连接，并没有发现有什么异常，只是程序方老反馈 redis 连接会出现异常断开的现象，经过仔细观察，发现出现问题的 pod 几乎都会落在同一个 node 节点上。于是从集群中剥离出该节点，进行排查。  

经过排查发现该 node 节点的 ```net.netfilter.nf_conntrack_tcp_timeout_established```  内核参数与其他节点不同，于是修成与其他节点相同，经过后期观察，发现问题解决。  

查找资料，发现```net.netfilter.nf_conntrack_tcp_timeout_established``` 相当于 ip_conntrack 模块下面的一个配置参数，维护着 iptables 的状态（NEW，ESTABLIISHED，RELATED）。
![ip-conntrack](http://wyong.cn/images/blog/k8s/ip_conntrack.jpeg)
ip_conntrack 模块，主要负责追踪数据包状态。
