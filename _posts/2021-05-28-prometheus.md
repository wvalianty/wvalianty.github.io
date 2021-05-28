---
layout: post
title: prometheus 监控的一角
categories: kubernetes
description: prometheus
keywords: k8s prometheus 待补充
---

## 一、 安装
```
helm repo add stable https://charts.helm.sh/stable

helm pull stable/prometheus-operator

# 改一下 value.yaml 中的 namespace
```
注意需要修改存储和数据保存时间以及节点亲和性等。  

安装完成后，有产生四个 CRD 配置对象，后续的配置基本都是通过 ServiceMonitor 和 PrometheusRule 来实现的。
* Prometheus：定义 operator 创建的 prometheus 集群
- Alertmanager：定义 operator 创建的 alertmanager 集群
* ServiceMonitor：定义采集目标，目标 metric 在哪里，一般是定义哪个命名空间的哪个service，命名空间也可以是 any，采集的目标一般是pod，通过 service 进行的服务发现。不需要重启服务，operator 会自动将采集目标加入到 prometheus target 中。
- PrometheusRule：定义告警规则，operator 会自动发现定义的 PrometheusRule。

![prometheus-operator-architecture](http://wyong.cn/images/blog/k8s/prometheus/prometheus-operator-architecture.jpeg)

## 二、 采集
监控数据采集是通过 ServiceMonitor 来定义的，通常需要配置采集哪个命名空间，哪个 pod 的哪个 接口。这里目标 pod 也是通过 k8s 中的 service 服务发现来找到的。配置案例如下：
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: autoscale-demo
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-operator-9.3.2
    heritage: Helm
    release: prometheus-operator
  name: autoscale-demo
  namespace: monitor
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
配置完成后就可以在 prometheus 看到监控目标。


## 三、 告警和通知
* 将告警规则定义在 prometheusrule，prometheus-operator 会自动加载。
* 通知链路信息配置在 alertmanager-prometheus-operator-alertmanager 这个 secret 里。

另外也可以在 grafana 中定义通知，相比通过 alertmanager 缺少延时、去重等功能，需要自己编写程序过滤告警。

## 四、 promSQL

#### 1、查询结果类型
* 瞬时向量,一组时序，每个时序只有一个采样值  

```
sum(http_requests_total) by (endpoint)

Element Value
{endpoint="http-metrics"}   4172549
{endpoint="https"}  963202
```

* 区间向量,一组时序，每个时序有多个采样值  

```
http_requests_total[2m]

Element Value
http_requests_total{code="200",endpoint="http-metrics",handler="prometheus",instance="192.168.130.78:10255",job="kubelet",method="get",namespace="kube-system",node="cn-beijing.i-2ze5l3vnrapyrb3ulntu",service="kubelet"}  321105 @1562142485.538
321108 @1562142515.538
321111 @1562142545.538
321114 @1562142575.538
http_requests_total{code="200",endpoint="http-metrics",handler="prometheus",instance="192.168.130.79:10255",job="kubelet",method="get",namespace="kube-system",node="cn-beijing.i-2ze5l3vnrapyrh0xxtsv",service="kubelet"}  321029 @1562142492.82
321032 @1562142522.82
321035 @1562142552.82
321038 @1562142582.82
```

* 标量数据,一个浮点数

#### 2、操作符
* 写在{}里面的“=，!=，=，!”
* 对返回的值有基本的加减乘除、比较类判断、逻辑类判断
* ```offset 1d```时间偏移

#### 3、向量匹配
一个向量是由指标名、标签、指标值三个部分组成，标签通常表示同一个指标的不同维度，在针对查询结果进行运算的时候，标签拥有非常重要的地位。  

在进行向量匹配运算的时候，使用``` on ```或``` ignoring ```平衡标签，使用``` group_left ``` 或 ``` group_right ```标明哪一侧为 *many*。  

**例如有下面采集数据**
```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

**one to one**  
```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
#----------------
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
#----------------
```

**one to many**  
```
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
#-------------------
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

## 五、 grafana
#### 1、大盘变量定义
在制作 grafan 大盘的时候，可以通过定义变量来展现不同的效果，比如定义一个查询类型的变量采集 pod 名字，然后在看大盘的时候就可以通过切换 pod 名字来查看不同的 pod 信息。具体定义可以通过``` label_values ```来进行定义，调试可以通过 prometheus 页面来调。
![grafana-variable](http://wyong.cn/images/blog/k8s/prometheus/grafana-variable.png)


## 六、 调试技巧
* kubefwd，在调试服务的时候，通常使用```kubectl port-forward```来将某个 pod/service 来映射到本地，但是当需要调试多个的时候，每个单独执行不方便，可以使用 [kubefwd](https://github.com/txn2/kubefwd) 映射整个命名空间。

## 参考
[全手动部署prometheus-operator监控Kubernetes集群遇到的坑](https://www.servicemesher.com/blog/prometheus-operator-manual/)
[Prometheus一条告警是怎么触发的](https://www.jianshu.com/p/af0f98fe7699)  
[alertmanager配置文件说明](https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_conf.html)  
[Prometheus 非官方中文手册](https://www.bookstack.cn/read/prometheus-manual/prometheus-querying-functions.md)  
[kubectl top 命令解析](https://cloud.tencent.com/developer/article/1645042)



