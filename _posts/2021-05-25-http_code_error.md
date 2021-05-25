---
layout: post
title: 运维常见 http 错误状态码 
categories: 故障排查
description: 运维常见 http 错误状态码
keywords: http 499 502 503 504
---
#### 1、 400 bad request
http报头，主机头不对。

#### 2、403 forbidden
可能因为路径存在，但是找不到 index.html 之类的。

#### 3、404 
主机 host 字段不对。
```
proxy_set_header Host wyong.cn;
```
#### 4、405 method not allow
请求方法错误。
#### 5、426 需要升级版本协议
在 istio 中遇到过这个错误，需要升级 http 协议版本。
```
proxy_http_version 1.1;
```
#### 6、499
客户端主动断开，数据返回太慢，客户端看不到，在 nginx 中会发现有 499 报错。

#### 7、500 internet server error
一般程序代码报错，短期解决问题可以试试重启，永久解决问题需要研发修改。

#### 8、502 bad gateway
一般为后端程序宕机了，监听端口已经不存在了。

#### 9、503 temporarily unavailable
一般后端太繁忙了，处理不过来了。

#### 10、504 gateway timeout
一般为 nginx 到后端真实服务距离太远了，nginx 请求后端超时。
可能 proxy_pass 后端是公网，nginx 到 proxy_pass 链路返回太慢，但服务没有宕机。
