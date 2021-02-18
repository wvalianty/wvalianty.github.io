---
layout: post
title: 准入控制器
categories: kubernetes
description: 准入控制器，动态准入控制器，MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。
keywords: k8s MutatingAdmissionWebhook ValidatingAdmissionWebhook
---
## 一、 准入控制器基础
#### 1、 控制器概念
准入控制器是一段代码，工作在访问 apiserver 的过程中，它会在请求通过认证和授权之后、对象被持久化之前拦截到达 apiserver 服务器的请求。通过修改提交的 yaml 文件，实现了为 PVC 设置默认 storageclass、拒绝在正处于删除状态的命名空间下创建对象。

动态准入控制器是两个特殊的控制器，包含验证性质的控制器和修改性质的控制器。每个控制器包含 WebhookConfiguration 和 webhook 两部分内容。它允许用户在 k8s 中 定义 ValidatingWebhookConfiguration 和 MutatingWebhookConfiguration 对象，指明所调用的 webhook 地址，然后 apiserver 在处理请求过程中，向定义的 webhook 地址发起调用，并根据调用返回的结果进行拦截、过滤。

修改控制器可以修改被其接受的对象，验证控制器则不行。任何一个阶段的任何控制器拒绝了该请求，则整个请求将立即被拒绝，并向终端用户返回一个错误。
![k8s-api-request-lifecycle](http://wyong.cn/images/blog/k8s/k8s-api-request-lifecycle.png)
#### 2、 
#### 3、 
#### 4、 

##### I、
## 五、 参考
## 六、 FAQ














2、查看默认启用
kube-apiserver -h


3、默认控制器事例

	AlwaysPullImages 修改每一个新创建的 Pod 的镜像拉取策略为 Always

	DefaultStorageClass  监测没有请求任何特定存储类的 PersistentVolumeClaim 对象的创建， 并自动向其添加默认存储类。
	
	NodeRestriction 该准入控制器限制了 kubelet 可以修改的 Node 和 Pod 对象 

	NamespaceLifecycle 该准入控制器禁止在一个正在被终止的 Namespace 中创建新对象，并确保 使用不存在的 Namespace 的请求被拒绝。



4、动态准入控制器


	设计思想启迪，运维有时间，所以代码并不次，


同时有的控制器也给人造成困惑，发现提交到 apiserver 的对象和预期的不一致，如果不知道控制器的存在，便感到很莫名其妙。



2、动态准入控制器
	validating
	mutating dryRun: true
		可以配置身份认证 client （apiserver） -> server（webhook）

	副作用&协调机制
		什么时候会有 ，手动配置 yaml还是自动添加的，是用来干啥的
		webhook 要求 具有 dryRun: true 字段的请求，必须不产生副作用，否则 API 请求失败    
		 kubectl create --dry-run=true -o yaml
		什么是副作用呢？把 yaml 改的不合法了？那么我只写 validating 不做修改，是不是不涉及呢？
		sideEffects
			some：api请求失败，并且不会掉用这个 webhook
			NoneOnDryRun：明确说明有 dryrun的没有副作用
	多次调用，幂等性质
	监控玩法
	审计注解
		event
		审计就是审计事件，那么 k8s 中审计事件是怎么回事？

	编写注意
		死锁，对现在可以成功运行的是否有影响
		副作用
		使用验证性的做最终状态确认
		延时，timeout
		版本
		幂等




	对应的配置，重点关键配置 -> 几个参数变化的影响

		思想借鉴，可以在配置文件定义请求检验地址，但是可能有 bug，于是有了 重点配置

3、调试工具
	telepresence

4、控制器的应用
	应用流量无损下线
	日后题？

5、控制器代码
	hello world 版本开始，然后开始迭代
	gorilla/mux 框架 web框架
	zap 日志
	cobra 命令行
	json-iterator json


	go 
		分包，包概念，包内处理，包对外输出
		怎么快速使用别人代码
		整齐代码的背后




参考
[官方 - 入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) 
[官方 - 动态准入控制](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#%E7%A1%AE%E4%BF%9D%E7%9C%8B%E5%88%B0%E5%AF%B9%E8%B1%A1%E7%9A%84%E6%9C%80%E7%BB%88%E7%8A%B6%E6%80%81) 
[代码编写参考](https://mritd.com/2020/08/19/write-a-dynamic-admission-control-webhook/)


