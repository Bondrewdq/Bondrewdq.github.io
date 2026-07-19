---
title: Ingress
tags: []
id: '274'
categories:
  - - k8s 实验
date: 2026-03-13 06:47:17
---

**Ingress 是 Kubernetes 的七层（HTTP/HTTPS）流量入口控制器**，通过**单一 IP/域名**将外部请求**路由到不同 Service**，替代多个 LoadBalancer 节省成本。

![](/wp-uploads/2026/03/1773383407-image.png)

* * *

**实验要求**：

*   如下创建新的 Ingress 资源：
    *   名称： echo
    *   Namespace： sound-repeater
    *   使用 Service 端口 8080 在 http://example.org/echo 上公开 echoserver-service Service。
*   可以使用以下命令检查 echoserver-service Service 的可用性，该命令应返回 Hello World ^\_^：
    *   candidate@master01:~$ curl http://example.org/echo

参考链接： [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)  
难度： ⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773383888-image.png)

_Ingress 是 Cluster 级别的资源，而非 Namespace 级别的_

### 1\. 查询ingressClassName

![](/wp-uploads/2026/03/1773383937-image.png)

### 2\. 创建 ingress

![](/wp-uploads/2026/03/1773383955-image.png)

### 3\. 创建并验证

![](/wp-uploads/2026/03/1773383973-image.png)

* * *

### 补充内容

#### Ingress Annotations

> 一句话总结
> 
> 在这个例子中，`rewrite-target: /` 让后端服务收到 `/` 而不是 `/echo`，确保后端能正确处理请求

`nginx.ingress.kubernetes.io/rewrite-target: /` 是 Nginx Ingress Controller 专用的配置。

_Annotations 翻译为“注解”实际上并不妥当，它不是类似于编程语言的注释，而是给第三方插件阅读的_

![](/wp-uploads/2026/03/1773384068-image.png)

##### 🔄 作用：URL 重写

原因

说明

**后端路径不匹配**

后端服务可能只监听 `/`，不识别 `/echo`

**路径隐藏**

不想让后端知道前端的路径结构

**统一入口**

多个 Ingress 路径指向同一个后端根路径

##### ⚠️ 为什么需要？

原因

说明

**后端路径不匹配**

后端服务可能只监听 `/`，不识别 `/echo`

**路径隐藏**

不想让后端知道前端的路径结构

**统一入口**

多个 Ingress 路径指向同一个后端根路径

##### 📝 常见 Nginx Ingress 注解

注解

作用

`nginx.ingress.kubernetes.io/rewrite-target`

URL 重写

`同上/ssl-redirect`

强制 HTTPS

`同上/proxy-body-size`

请求体大小限制

`同上/rate-limit`

限流

* * *