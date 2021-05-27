---
layout: post
title: istio
categories: kubernetes
description: istio 使用经验
keywords: istio virtualservice gateway destinationrule
---
一个代理能做什么，作为一个熟练使用 nginx 的人，觉得代理也就是正反向代理、负载均衡的功能。然而 envoy 得让我用一个新的视角重新去理解代理。envoy 首先通过 iptables 劫持流量，将所有流量交给 envoy 这个代理进行管控。这样所有流量都可以通过 envoy 来进行控制，再加上 istio 实现的控制面板，这样就实现了一个全新的服务治理框架，便有微服务治理服务网格。

## 一、 论服务架构
随着业务的扩展，单体应用肯定是无法承载的。需要对服务进行拆分，在具体执行服务的拆分的时候，主要在两个方向上进行拆分，一个是垂直方向，针对功能，比如可以将原来的多个功能拆分成不同的模块，另一个是水平扩展，比如原来部署一台服务器运行服务，现在可以不熟两台服务同时运行此服务，然后在前面加上一个负载均衡。这两点构成了服务拆分的主要依据，并依此扩展出了微服务架构。  

在垂直方向针对服务功能进行拆分，被称为面向服务的拆分（SOA），将原来功能耦合在一起的服务分散成多个功能单元，多个单元协同工作，各个应用程序之间可以使用 RPC 调用，具体 RPC 协议有 GRPC。拆分依据的功能可以是根据不同功能进行路由，可以根据用户 uid 等属性进行路由等。  

在水平方向的扩展通常被称为分布式架构（DSA），有很多去中心化服务框架、负载均衡均衡等都实现了服务在水平方向的扩展。水平扩展通常由多个相同的单体来提高吞吐量。  

针对业务在水平和垂直方向进行更细粒度的切分，就形成了微服务（MSA）。切分成微服务之后，会面临着大量的微服务需要管理，这给业务维护工作也带来了挑战。面对着诸多的微服务，诸多的服务和链路，服务网格就出现了，服务网格通过代理来管控业务流量，再通过控制面板针对每个代理进行统一控制，解决了微服务复杂的网络链路。  
 
## 二、 组件及架构
istio 有很多组件，可以针对不同环境、不同的功能需求进行选择安装。  
![k8s-service](http://wyong.cn/images/blog/k8s/istio/istio-architecture.png)

理解 istio 的架构，主要从两个维度理解，一个是控制平面，一个是数据平面。控制平面就是下发策略下发配置链路，数据平面就是流量链路。

### 1、pilot
pilot 主要有两个功能  
* 服务发现，pilot 可以从 k8s 中提取配置信息。
* 动态下发配置，通过 XDS 协议动态配置代理，动态修改 envoy 代理的转发策略，envoy 可以支持动态生效。

![k8s-service](http://wyong.cn/images/blog/k8s/istio/istio-pilot.png)

### 2、istio-telemetry
telemetry 主要功能是收集上报的链路状态信息，然后其他组件可以针对上报的链路信息进行制图，形成便于观测的链路状态图。

![k8s-service](http://wyong.cn/images/blog/k8s/istio/istio-telemetry.png)

### 3、istio-policy
istio-policy 主要负责下发访问策略，可以配置服务访问的黑白名单等。

![k8s-service](http://wyong.cn/images/blog/k8s/istio/istio-policy.png)

### 4、citadel
istio 中的证书 CA 角色，负责证书的生成、颁发、轮换、撤销。
![k8s-service](http://wyong.cn/images/blog/k8s/istio/citadel.png)

### 5、galley
galley 主要校验下发的配置信息的正确性，只工作在控制平面。

### 6、envoy gateway 等代理
envoy 是数据平面的执行者，作为代理的身份，envoy 整个服务网格中的很多位置，包括网格内部流量转发和网格边缘流量转发。  

envoy 程序本身是单进程多线程的工作模型，性能可以媲美 nginx。envoy 程序属于线程模型，其中主线程负责 envoy 的启动关闭，XDS 调用处理、运行时配置等，相关所有事件都是异步非阻塞。在一条请求的处理过程中，在 envoy 中经历多个处理阶段，包括 tcp 处理，http 处理。

![k8s-service](http://wyong.cn/images/blog/k8s/istio/istio_ingressgateway.png)

### 7、sidecar-injector 
针对配置自动注入的 pod，自动注入 initContainers 和 envoy 代理 sidercar。如果想让某一个命名空间的 pod 自动注入，则给该命名空间标记 ```istio-injection: enabled``` 。

### 8、kiali jaeger 
istio 的线路观察工具，kiali 可以形成请求链路图并且可以展示链路状态，QPS 等信息。jaeger 可以展示详细的请求返回时长等。

## 三、 配置对象
istio 组件诸多，但是经常配置的对象只有 gateway、virtualservice、destinationrule 这三个 CRD 对象。
### 1、gateway
经常使用的 gateway 一般是工作在网格边缘的，监听在配置的域名和端口上，将流量接入，如果没有在 gateway 中匹配到对应的域名，请求将返回 404。  
gateway 配置案例
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: internet-gateway
spec:
  selector:
    app: istio-ingressgateway
  servers:
  - hosts:
    - aaa.XXX.COM
    - bbb.XXX.COM
    port:
      name: https-tls-X
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/XXX.COM.key
      serverCertificate: /etc/istio/ingressgateway-certs/XXX.COM.pem
  - hosts:
    - aaa.YYY.COM
    - bbb.YYY.COM
    port:
      name: https-tls-Y
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/YYY.COM.key
      serverCertificate: /etc/istio/ingressgateway-certs/YYY.COM.pem
  - hosts:
    - aaa.YYY.COM
    - bbb.XXX.COM
    port:
      name: http
      number: 80
      protocol: HTTP
    tls:
      httpsRedirect: true
```

上面配置了 **两个域名的 https，并且 http 自动跳转到 https**。其中证书是放在 ``` istio-system ```命名空间下的``` istio-ingressgateway-certs ``` 这个``` secret ```下面，而且 **这个 ``` secret ```的名字是固定**的。
```
apiVersion: v1
data:
  XXX.COM.key: 
  XXX.COM.pem: 
  YYY.COM.key: 
  YYY.COM.pem: 
# 正常 pem 证书和 key 原文 base64 编码后
kind: Secret
metadata:
  name: istio-ingressgateway-certs
  namespace: istio-system
type: Opaque
```

可以通过下面命令查看证书是否挂在成功。
```
kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-certs
```

### 2、virtualservice
virtualservice 代表虚拟服务，有网格内和网格边缘，工作的网格边缘的，gateway 接收完流量转发到此 virtualservice。配置有``` spec.gateways ```字段的默认是工作在网格边缘，加上``` mesh ```表示同时支持网格内和网格边缘。  
配置案例
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: XXX-COM
spec:
  gateways:
  - internet-gateway-https
  hosts:
  - XXX.COM
  http:
  - match:
    - uri:
        prefix: /xxx/
# 这个重写不是跳转，是在 ingressgateway 向后段发送请求的时候替换路径。
    rewrite:
      uri: /
    route:
    - destination:
        host: server1
        port:
          number: 80
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: server2
        port:
          number: 80
  - match:
    - uri:
        regex: /([a-z][a-z]_[A-Z][A-Z]|xxx/?$
    route:
    - destination:
        host: server3
        port:
          number: 80
```

### 3、destinationrule
可以用来和``` virtualservice ```一起作灰度发布。 

### 4、使用经验
建议配置 istio-ingressgateway  deployment 的 replicas 和 node 节点数量一致，并配置 pod 反亲和性。

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

## 参考
[istio官方文档](https://istio.io/latest/zh/docs/setup/)  
[Benchmarking 5 Popular Load Balancers: Nginx, HAProxy, Envoy, Traefik, and ALB](https://www.loggly.com/blog/benchmarking-5-popular-load-balancers-nginx-haproxy-envoy-traefik-and-alb/)  

