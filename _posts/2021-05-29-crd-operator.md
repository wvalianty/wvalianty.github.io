---
layout: post
title: 快速编写一个自定义控制器 CRD
categories: kubernetes
description: crd controller
keywords: k8s operator crd controller
---
Kubernetes Operator 的教程颇多，但是对于新手可能不够友好，我这个博客争取可以让新手更容易在本地运行起来，花费更短的时间，不需要梯子就能运行起来。当然生产环境还是使用官方正宗的代码吧。  

### 1、环境配置

* go version go1.11.4 darwin/amd64 [下载链接](https://golang.org/doc/install?download=go1.11.4.darwin-amd64.pkg)

* operator-sdk version: v0.7.0 [下载链接](https://github.com/operator-framework/operator-sdk/releases?after=v0.13.0)


### 2、创建项目
*  创建项目目录拷贝依赖
```
☁  wvalianty  operator-sdk new opdemo2
INFO[0000] Creating new Go operator 'opdemo2'.          
INFO[0000] Created cmd/manager/main.go                  
INFO[0000] Created build/Dockerfile                     
INFO[0000] Created build/bin/entrypoint                 
INFO[0000] Created build/bin/user_setup                 
INFO[0000] Created deploy/service_account.yaml          
INFO[0000] Created deploy/role.yaml                     
INFO[0000] Created deploy/role_binding.yaml             
INFO[0000] Created deploy/operator.yaml                 
INFO[0000] Created pkg/apis/apis.go                     
INFO[0000] Created pkg/controller/controller.go         
INFO[0000] Created version/version.go                   
INFO[0000] Created .gitignore                           
INFO[0000] Created Gopkg.toml                           
INFO[0000] Run dep ensure ... 
INFO[0000] Creating new Go operator 'opdemo2'.          
INFO[0000] Created cmd/manager/main.go                  
INFO[0000] Created build/Dockerfile                     
INFO[0000] Created build/bin/entrypoint                 
INFO[0000] Created build/bin/user_setup                 
INFO[0000] Created deploy/service_account.yaml          
INFO[0000] Created deploy/role.yaml                     
INFO[0000] Created deploy/role_binding.yaml             
INFO[0000] Created deploy/operator.yaml                 
INFO[0000] Created pkg/apis/apis.go                     
INFO[0000] Created pkg/controller/controller.go         
INFO[0000] Created version/version.go                   
INFO[0000] Created .gitignore                           
INFO[0000] Created Gopkg.toml                           
INFO[0000] Run dep ensure ... 
```
运行到这里直接 ctrl + C 结束，执行 ```cp -r opdemo/vendor opdemo2/```。因为代码运行到这里开始拉取依赖的代码库，这里需要科学，而且代码量比较大，需要等待好久，所以我用了更简单的方法，直接复制别人的 ```vendor``` ，需要注意版本的不同。

* 添加控制器
```
☁  opdemo2  operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService
INFO[0000] Generating controller version app.example.com/v1 for kind AppService. 
INFO[0000] Created pkg/controller/appservice/appservice_controller.go 
INFO[0000] Created pkg/controller/add_appservice.go     
INFO[0000] Controller generation complete.   
```

* 编写 CRD 对象  
在下面文件中配置自定义对象 CRD 的 spec
```
github.com/wvalianty/opdemo2/pkg/apis/app/v1/appservice_types.go
```

* 生成代码
```
☁  opdemo2  operator-sdk generate k8s
INFO[0001] Running deepcopy code-generation for Custom Resource group versions: [app:[v1], ] 
INFO[0003] Code-generation complete. 
```


### 3、编写控制逻辑
自定义 CRD 对象的控制逻辑在```github.com/wvalianty/opdemo2/pkg/controller/appservice/appservice_controller.go```中的```Reconcile```方法里。

控制器通过不断的 list/ watch CRD 对象，当 CRD 对象发生变化，则调用```Reconcile```方法，但是 CRD 的子对象发生变化，则不会触发掉用```Reconcile```方法。比如上面的``` appservice```对象发生变化会触发调用```Reconcile```方法，但是当``` appservice```创建出来的```service```或```deployment```等对象发生变化则不会触发```Reconcile```方法。

### 参考

[Kubernetes Operator 快速入门教程
](https://www.qikqiak.com/post/k8s-operator-101/)  
