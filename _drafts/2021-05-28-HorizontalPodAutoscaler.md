---
layout: post
title: HPA 
categories: kubernetes
description: HPA
keywords: k8s HPA 
---

## 一、kubernetes 中的扩容

## 二、HPA 类型
保守策略

### 1、cpu
### 2、内存
### 3、自定义指标
prometheus-adapter

## 三、实践

## 参考


## 二、HPA 类型
hpa 的自定义类型（custom），meitric 主要通过 adapter 实现，adapter 通过采集 prometheus 中的数据，将自身接口注册到 apiserver aggregator，从而实现可以通过 apiserver 查询到 metric 数据，所以第一步就是要配置好 adapter。(configmap)

```
apiVersion: v1
data:
  config.yaml: |
    rules:
    - seriesQuery: '{__name__=~"^container_.*",container!="POD",namespace!="",pod!=""}'
      seriesFilters: []
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: ^container_(.*)_seconds_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>,container!="POD"}[5m]))
        by (<<.GroupBy>>)
    - seriesQuery: '{__name__=~"^container_.*",container!="POD",namespace!="",pod!=""}'
      seriesFilters:
      - isNot: ^container_.*_seconds_total$
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: ^container_(.*)_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>,container!="POD"}[5m]))
        by (<<.GroupBy>>)
    - seriesQuery: '{__name__=~"^container_.*",container!="POD",namespace!="",pod!=""}'
      seriesFilters:
      - isNot: ^container_.*_total$
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: ^container_(.*)$
        as: ""
      metricsQuery: sum(<<.Series>>{<<.LabelMatchers>>,container!="POD"}) by (<<.GroupBy>>)
    - seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
      seriesFilters:
      - isNot: .*_total$
      resources:
        template: <<.Resource>>
      name:
        matches: ""
        as: ""
      metricsQuery: sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)
    - seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
      seriesFilters:
      - isNot: .*_seconds_total
      resources:
        template: <<.Resource>>
      name:
        matches: ^(.*)_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)
    - seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
      seriesFilters: []
      resources:
        template: <<.Resource>>
      name:
        matches: ^(.*)_seconds_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)
    - seriesQuery: '{__name__=~"^Result.*",namespace="monitor"}'
      seriesFilters: []
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: ^Result.*$
        as: "http_requests_per_second"
      metricsQuery: <<.Series>>{<<.LabelMatchers>>}
kind: ConfigMap
metadata:
  labels:
    app: prometheus-adapter
    chart: prometheus-adapter-2.1.1
    heritage: Tiller
    io.cattle.field/appId: prometheus-adapter
    release: prometheus-adapter
  name: prometheus-adapter
  ```
###  adapter 采集配置注意
 * seriesQuery 指定需要处理的Prometheus的metrics
 * resources 设置metric与kubernetes resources的映射关系
 * name 重命名
 * metricsQuery 目标值的具体计算公式，注意第一个 seriesQuery 采集到的仅仅是目标 metric，不进行计算
### 查看 adapter 配置是否正常
* 自定义指标查看
1. 先在 prometheus 中查
2. 再查 adapter 注册到 apiserver 的接口
```

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/monitor/pods/*/Result:"|jq .
```

3. 顺便看 metric 接口

```
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/monitor/pods/alertmanager-prometheus-operator-alertmanager-0"|jq .
```


## 三、实践
### 3.1 配置和调试
* v1 和 v2 版本可以共存, v1 需要deployment request，v2 不需要
* 多个指标可以共存
* 扩容公式 ```desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]```
* metric 采集不到，保守算法
* **快速扩容，低速缩容，网易云参考不错**

#### 3.1.1 老版本，仅支持 CPU，但是需要配置 request
一条命令创建  

```
kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=10
```

案例  

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: ysab
  namespace: monitor
  resourceVersion: "6370145"
  uid: 0715fdd6-4c36-4eb5-a3f1-398533c6279b
spec:
  maxReplicas: 3
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ysab
  targetCPUUtilizationPercentage: 10
```

#### 3.1.2 resource

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: python-response
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 90Mi
      #targetAverageUtilization: 60
```

#### 3.1.3 自定义 metric，一般是 pod

```
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-app
spec:
  scaleTargetRef:
    # point the HPA at the sample application
    # you created above
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  # autoscale between 1 and 10 replicas
  minReplicas: 1
  maxReplicas: 3
  metrics:
  # use a "Pods" metric, which takes the average of the
  # given metric across all pods controlled by the autoscaling target
  - type: Pods
    pods:
      # use the metric that you used above: pods/http_requests
      metricName: "Result:"
      # target 500 milli-requests per second,
      # which is 1 request every two seconds
      #Value: 500m
      metricName: "Result:"
      targetAverageValue: 66666671666568m
```

### 3.2 留点心
如果数据采集不到，hpa 控制器会怎么处理，如果扩了 pod，但是 pod 一直处于非 ready 状态。hpa 的保守机制是怎么处理的呢？扩容是怎么处理的？缩容是怎么处理的？

### 3.3 实践

1. 部署采集目标  

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - image: marklaioffer/autoscale-demo
        name: metrics-provider
        ports:
        - name: http
          containerPort: 8080
```

2. 暴露采集目标  

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sample-app
  name: sample-app
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: sample-app
  type: ClusterIP
```

3. 添加采集目标  

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: autoscale-demo 
  labels:
    app: autoscale-demo 
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-operator-9.3.2
    heritage: Helm
    release: prometheus-operator
spec:
  endpoints:
  - path: /metrics
    port: http 
  namespaceSelector:
    matchNames:
    - monitor
  selector:
    matchLabels:
      app: sample-app 
```

4. 在 prometheus target 查看  

5. 查看 adapter  

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/monitor/pods/*/Result:"|jq 
```

6. 创建 hpa  

```
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-app
spec:
  scaleTargetRef:
    # point the HPA at the sample application
    # you created above
    apiVersion: apps/v1
    kind: Deployment
    name: sample-app
  # autoscale between 1 and 10 replicas
  minReplicas: 1
  maxReplicas: 3 
  metrics:
  # use a "Pods" metric, which takes the average of the
  # given metric across all pods controlled by the autoscaling target
  - type: Pods
    pods:
      # use the metric that you used above: pods/http_requests
      metricName: "Result:" 
      # target 500 milli-requests per second,
      # which is 1 request every two seconds
      #Value: 500m
      metricName: "Result:"
      targetAverageValue: 66666671666568m 

```

7. 调整目标值，查看扩缩  

## 参考
[网易云](https://zhuanlan.zhihu.com/p/245208287)  
[阳明](https://www.qikqiak.com/post/k8s-hpa-usage/)  
[使用k8s-prometheus-adapter实现HPA](https://www.cnblogs.com/charlieroro/p/11898521.html)  
[知乎-探索Kubernetes HPA](https://zhuanlan.zhihu.com/p/89453704)  

