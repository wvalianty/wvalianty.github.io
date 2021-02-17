---
layout: post
title: Kustomize 
categories: kubernetes
description: 我对 kustomize 的总结
keywords: k8s kustomize 
---
## 一、 什么是 kustomize ？
k8s 实现了运维基础设施的可编排，实现了通过 yaml 文件来编排应用的基础设施。随之而来的是大量基础设施编排文件 yaml 的管理。kustomize 从多环境、多项目的角度出发，针对不同项目、不同环境之间的差异，通过修补 yaml 文件的方式组织和抽象 yaml 文件，让 yaml 基础设施编排文件的管理变得更加简单。

kustomize 的主要功能就是 **patch**。但这个 patch 相对来说比较广义，patch 的对象可以是来自 yaml 文件中的 k8s 对象（deployment spec labels），也可以是不同文件夹之间的 patch，还可以是单纯的组织两个 yaml 文件。

从 1.14 版本开始，kubectl 直接内置支持了 kustomize。

## 二、 基础入门
#### 1、 kustomization.yaml 介绍
kustomization.yaml 是 kustomize 的配置文件，需要修补的资源对象、修补具体内容都是定义在此文件中。

#### 2、 kustomization.yaml 语法
* resources
定义修补的源文件
* commonAnnotations
给修补的源文件中所有对象都添加的声明
* commonLabels
给修补的源文件中所有对象都添加的标签
* namePrefix
给修补的源文件中所有对象都设置的名字前缀
* nameSuffix
给修补的源文件中所有对象都设置的名字后缀
* namespace
给修补的源文件中所有对象设置安装的命名空间
* patchesStrategicMerge
设置补丁文件，源文件可能有多个对象，补丁文件修补是根据名字修补的
* bases
定义源文件夹，类似上面的 resources，但是这里的是文件夹，文件夹内包括 kustomization.yaml 文件

#### 3、常用命令
* 预览生成的 yaml 文件
```
k kustomize .
# 或使用 kustomize 命令
kustomize build .
```

* 执行执行安装 
```
kubectl apply -k .
```

#### 4、注意
源文件如果包含多个 k8s 对象，需要使用“---”间隔开

## 三、 实例 
#### 1、 组织文件
```
# 当前文件夹结构
☁  demo-zuzhi  tree .
.
├── deployment.yaml
├── kustomization.yaml
└── service.yaml

0 directories, 3 files
==========================================
☁  demo-zuzhi  cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
==========================================
☁  demo-zuzhi  cat service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
==========================================
# 使用 kustomize 组织使用哪个文件来部署
☁  demo-zuzhi  cat kustomization.yaml
resources:
- service.yaml
```
#### 2、 patch 文件
```
☁  demo_file  cat patch_kv_cm.yaml
apiVersion: v1
data:
  a: aa
  b: bb
  c: dd
kind: ConfigMap
metadata:
  name: kv-cm
☁  demo_file  cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- kv-cm.yaml
patchesStrategicMerge:
- patch_kv_cm.yaml
commonLabels:
  env: dev
nameSuffix: "-001"
namespace: "default"
☁  demo_file  k kustomize .
apiVersion: v1
data:
  a: aa
  b: bb
  c: dd
kind: ConfigMap
metadata:
  labels:
    env: dev
  name: kv-cm-001
  namespace: default
```
#### 3、 patch 文件夹
如果修补对象是文件夹，带修补的源文件夹的官方称呼为基准，作为补丁的文件夹叫做覆盖。
```
#  当前文件夹结构，其中 base 为待修补的源文件夹，dev 和 prod 分别表示测试和生产
☁  demo-dir  tree .
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── dev
│   └── kustomization.yaml
└── prod
    └── kustomization.yaml

3 directories, 5 files
==========================================
☁  demo-dir  cat base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
==========================================
☁  demo-dir  cat base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
==========================================
☁  demo-dir  cat base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
==========================================
☁  demo-dir  cat dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
==========================================
☁  demo-dir  cat prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
```
这里在 dev 文件里面定义 kustomization.yaml，指定了修补的源文件夹
```
☁  demo-dir  k kustomize ./dev
apiVersion: v1
kind: Service
metadata:
  labels:
    run: my-nginx
  name: dev-my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
```
## 四、 与 helm 的对比 
helm 更加偏向于单一项目的管理，抽象出一个项目中的配置。可以很好的进行一个项目的依赖管理、生命周期管理、定制化等。

而 kustomize 更倾向于差异化编排。对于多环境、多项目的场景更加合适，可以更加专注于差异，更加专注于变动，减少重复性的工作。

helm 和 kustomize 也可以结合起来使用，先利用 helm 生成 yaml 文件作为 kustomize 的源文件，再利用 kustomize 去修补。
```
helm template demo . > demo_helm_templated.yaml
# 在 kustomization.yaml 指定 demo_helm_templated.yaml 作为 resources
☁  demo01  cat kustomization.yaml
resources:
- deployment.yaml
commonLabels:
  app: bingo
```

## 五、 参考
[官方文档](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/)  
[CNCF - 阳明](https://mp.weixin.qq.com/s/1F5OUoNB8P25dtvz9kuklA)

## 六、 FAQ