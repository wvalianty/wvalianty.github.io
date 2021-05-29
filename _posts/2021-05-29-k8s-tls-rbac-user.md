---
layout: post
title: k8s 鉴权 
categories: kubernetes
description: k8s rbac adduser node节点鉴权
keywords: k8s rbac adduser node节点鉴权 
---
## 一、 
#### 1、
##### I、 
## 五、 参考 
## 六、 FAQ


1、证书关键属性
CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
C: Country， 国家
L: Locality，地区，城市
O: Organization Name，组织名称，公司名称
OU: Organization Unit Name，组织单位名称，公司部门
ST: State，州，省



k8s 证书  服务端证书 客户端证书
etcd 单独一套 etcd CA 只被 apiserver
    节点间通讯  peer cert
    对外 restful


apiserver kube CA
    访问客户端有
        kubectl
        scheduler
        controller
        kubelet
        kube-proxy coredns ？
    apiserver 也会访问 kubelet  这个证书是 kubelet 自动生成的，是根据 kube CA 生成的  证书链 kubelet ca 是 kube CA 的二级 ca


三套 ca
    etcd 作为服务端
    apiserver 作为服务端
    前端代理 作为服务端

    还有一种，apiserver 请求 kubelet，ca 是 每个 kubelet 自己自动生成，这个 ca 是 apiserver 的二级 ca，因为 apiserver 信任了一级 ca，信任根 ca，就信任 根 ca 下面的 二级 ca

    向下信任，不是向上信任
    上可以公示下，信任上，就信任它所有的下


front-proxy-ca
前端代理 ca，为了校验自定义代理扩展程序
这里起码可以看一下 metric server，它属于 前端代理
扩展的是资源，是一个动态注册，注册到 aggrate

资源 和 webhook










kubelet 如何注册到 kubernetes 集群
kubelet 首先读取配置文件 bootstrap，寻找到 master 地址，申请加入
bootstrap 里面配置着 master url 和 kubelet 启动的密钥，与 master 沟通会产生一个令牌，后面就使用令牌进行通讯
kubelet 需要申请加入 master，是通过申请签名的方式，kubelet 先在 集群里面创建 csr 请求
当然 kubelet 创建这个 csr 是需要有对应权限的，这个权限是通过 rbac 进行的，k8s 集群默认有些 clusterrole 和 rolebinding，那么 kubelet 是怎么和权限对应的呢？
识别 kubelet 是通过 证书中的 CN（名称） 和 O（组）确定的
确定了kubelet 的组，就确定了 kubelet 的权限
所以通常的要求是，kubelet 申请的证书，组要在 “system:nodes”，名称需要是 “system:node:<nodeName>”，因为 system:nodes 组之外的 kubelet 不会被 Node 鉴权模式授权，kubelet 最开始启动需要到 apiserver 请求签证，需要创建的资源是 csr（certificatesigningrequest），可以通过 kubectl get csr 来查看，可以手动签发，也可以让 controller-manager 自动签发，签发的命令是 kubectl certificate approve。
对应的 apiserver 启动要求开启 使用 node 鉴权， --authorization-mode=Node,RBAC。

kubelet、controller-manager、scheduler 启动配置文件都是 kubeconfig 文件。
controller-manager、scheduler 访问 apiserver 可以通过本地非安全的接口进行访问。
 


参考
TLS 启动引导
使用 Node 鉴权
 证书过期
kubelet 证书过期
kubeadm证书以及etcd证书过期处理
kubuadm 十年证书创建
修改kubeadm有效期
