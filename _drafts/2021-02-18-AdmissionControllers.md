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

动态准入控制器是两个特殊的控制器，包含验证性质的控制器和修改性质的控制器。每个控制器包含 **WebhookConfiguration** 和 **webhook** 两部分内容。它允许用户在 k8s 中 定义 ValidatingWebhookConfiguration 和 MutatingWebhookConfiguration 对象，指明所调用的 webhook 地址，然后 apiserver 在处理请求过程中，向定义的 webhook 地址发起调用，并根据调用返回的结果进行拦截、过滤。

修改控制器可以修改被其接受的对象，验证控制器则不行。任何一个阶段的任何控制器拒绝了该请求，则整个请求将立即被拒绝，并向终端用户返回一个错误。

验证准入控制器和修改准入控制器工作位置
![k8s-api-request-lifecycle](http://wyong.cn/images/blog/k8s/k8s-api-request-lifecycle.png)

我们可以定义自己的准入控制器，同时也需要注意有的控制器容易也给人造成困惑，当提交到 apiserver 的对象和预期的不一致，如果不知道控制器的存在，会很莫名其妙。
#### 2、 k8s 默认控制器示例
* AlwaysPullImages 修改每一个新创建的 Pod 的镜像拉取策略为 Always。

* DefaultStorageClass 监测请求中没有配置存储类的 PersistentVolumeClaim，为其配置添加存储类。
	
* NodeRestriction 限制了 kubelet 可以修改的 Node 和 Pod 对象。

* NamespaceLifecycle 该准入控制器禁止在一个正在被终止的 Namespace 中创建新对象，并确保 使用不存在的 Namespace 的请求被拒绝。

#### 3、 查看默认启用的准入控制器
可以在 apiserver 启动的时候指定启用哪些准入控制，但是一般情况不需要自己指定。

查看默认启动了哪些准入控制器
```sh
kube-apiserver -h
```
#### 4、 如何定义一个准入控制器
* 是使用验证性控制器还是使用修改性质控制器？  
当需要执行一些判断的时候，比如可以禁止在某个时间段内进行什么操作？禁止什么样的 spec 的对象创建？禁止什么操作指令，create or delete？这个时候需要使用的是验证性质准入控制器。  
在需要修对某一类对象进行一些 patch 操作的时候，应该使用修改性质控制器，比如我本地的 kuberntes 集群中不允许 deployment pod 重启策略 restartPolicy 为 Never，只能是 always。
* 创建准入控制器
创建一个控制器一共分两步，一步是告诉 k8s apiserver 你的控制器地址，另一步是编写程序处理具体的控制行为。

## 二、 动态准入控制器
#### 1、 定义一个验证性质准入控制器
```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validating-webhook.wyong.cn
webhooks:
- admissionReviewVersions:
  - v1
  - v1beta1
  clientConfig:
    caBundle: '填入对应的'
    service:
      name: test
      namespace: test
      path: /validating/check-deploy-time
      port: 443
  failurePolicy: Ignore
  matchPolicy: Equivalent
  name: validating-webhook.wyong.cn
  namespaceSelector:
    matchLabels:
      validating-webhook.wyong.cn: "true"
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    - DELETE
    resources:
    - pods
    - deployments
    scope: Namespaced
  sideEffects: None
  timeoutSeconds: 5
```
#### 2、 调试利器 telepresence
这里有一个调试技巧，可以使用 telepresence 将本地服务映射到 k8s 集群里面，这样不需要每次编写完代码都需要更新代码到 deployment。
#### 3、 编写一个 webhook
这里参考了一个大神的博客，参见[代码编写参考]。**运行中我发现了一个小问题，程序不能优雅的停止，[我的github](https://github.com/wvalianty/goadmission/)**，我修改了几行代码，使其可以优雅的停止。
#### 4、 编写注意事项

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




## 五、 参考
[官方 - 准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) 

[官方 - 动态准入控制](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#%E7%A1%AE%E4%BF%9D%E7%9C%8B%E5%88%B0%E5%AF%B9%E8%B1%A1%E7%9A%84%E6%9C%80%E7%BB%88%E7%8A%B6%E6%80%81) 

[代码编写参考](https://mritd.com/2020/08/19/write-a-dynamic-admission-control-webhook/)


## 六、 FAQ
4、控制器的应用
	应用流量无损下线
	日后题？


