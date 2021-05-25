---
layout: post
title: pod 调度总结 
categories: kubernetes
description: pod 调度总结 
keywords: pod scheduler 
---
pod 的调度方式有很多种，我这里作一下总结。
 
#### 1、 nodeSelector
最简单的调度方式，只能配置一个 pod 调度到存在标签的具体 node 节点上。
#### 2、 node 禁止调度
可以直接编辑 node，设置 ``` unschedulable: true ```，或者通过 ```kubectl cordon```,则该节点禁止有 pod 新调度过来。
#### 3、 污点和容忍
相比 nodeSelector 这种硬性要求，污点和容忍的调度机制提供了更为灵活的调度方式，可以配置倾向于调度到某些节点。还可以基于 NoExecute 来驱逐节点上面运行的pod。

tolerations 和taints 可以有多组，如果满足 tolerations 的 key 多于等于 taints 的，则可以调度。  

我的配置案例
节点配置
```
  taints:
  - effect: NoSchedule
    key: gitlabrunner
```
pod 配置
```
      tolerations:
      - effect: NoSchedule
        key: gitlabrunner
        operator: Exists
```
#### 4、 亲和性
相比污点的调度策略，亲和性提供了更为灵活的调度策略。亲和性针对调度策略方面考虑的更为全面，不仅考虑了 pod 与 node 之间的关系，而且还考虑了 pod 与 pod 之间的关系。

| nodeAffinity | podAffinity | podAntiAffinity |
| :-----: | :-----: | :------: |
| 配置pod倾向于调度到某node | Pod的多副本调度到相同的节点之上 | Pod的多副本调度到不同的节点之上 |

具体执行调度时，有硬调度和软  

| preferredDuringSchedulingIgnoredDuringExecution | requiredDuringSchedulingRequiredDuringExecution | requiredDuringSchedulingIgnoredDuringExecution |
| :-----: | :------: | :-----: |
| soft 目标节点最好能满足此条件 IgnoredDuringExecution 意味着，如果 Pod 已经调度到节点上以后 节点的标签发生改变，使得节点已经不再匹配该亲和性规则了，Pod 仍将继续在节点上执行 | 当节点的标签不在匹配亲和性规则之后 Pod 将被从节点上驱逐 | hard，目标节点必须满足此条件|


案例
```
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
            weight: 2
```

```
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - istio-ingressgateway
              topologyKey: kubernetes.io/hostname
            weight: 100
```

#### 5、 配置命名空间级别的调度
首先需要确定 apiserver 中已经支持 ```--enable-admission-plugins=PodNodeSelector```，然后可以配置一个命名空间下面的 pod 调度到指定的 node 上。
```
apiVersion: v1
kind: Namespace
metadata:
 name: namespace
 annotations:
   scheduler.alpha.kubernetes.io/node-selector: env=test
spec: {}
```
#### 6、 参考 
[Kubernetes中的亲和性](https://blog.csdn.net/jettery/article/details/79003562)  
[污点和容忍度](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/)  
[Kubernetes中的亲和性](https://blog.csdn.net/jettery/article/details/79003562)  
[How to assign a namespace to certain nodes](https://stackoverflow.com/questions/52487333/how-to-assign-a-namespace-to-certain-nodes)
