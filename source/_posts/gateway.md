---
title: Gateway
tags: []
id: '321'
categories:
  - - k8s 实验
date: 2026-03-14 06:40:27
---

K8s **Gateway** 是**官方新一代流量入口标准（Gateway API）**，用来**替代 / 升级传统 Ingress**，专门管理**外部流量进入 Kubernetes 集群**的南北向流量。

**Gateway = 更强大、更规范、云原生的下一代 Ingress**

* * *

**实验要求**：

*   将现有 Web 应用程序从 Ingress 迁移到 Gateway API。您必须维护 HTTPS 访问权限。
    *   _注意：集群中安装了一个名为 nginx 的 GatewayClass 。_
*   首先，创建一个名为 `web-gateway` 的 Gateway ，主机名为 `gateway.web.k8s.local` ，并保持现有名为 `web` 的 Ingress 资源的现有 TLS 和侦听器配置。
*   接下来，创建一个名为 `web-route` 的 `HTTPRoute` ，主机名为 `gateway.web.k8s.local` ，并保持现有名为 `web` 的 Ingress 资源的现有路由规则。
    *   _您可以使用以下命令测试 Gateway API 配置：_
        *   \[candidate@cka000057\]$ curl -Lk https://gateway.web.k8s.local:31443
*   最后，删除名为 `web` 的现有 Ingress 资源。

参考链接： [https://kubernetes.io/docs/concepts/services-networking/gateway/](https://kubernetes.io/docs/concepts/services-networking/gateway/)  
难度：⭐⭐⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773469142-image.png)

`kubectl get ingress web -o yaml`：

*   创建 `Gateway` 时需要： 查看 `spec.tls.secretName`
*   创建 `HTTPRoute` 时需要：查看 `spec.rules.http.paths.backend` 和 `spec.rules.http.paths.path`

![](/wp-uploads/2026/03/1773469169-image.png)

### 1\. 编写 gateway.yaml 并应用

![](/wp-uploads/2026/03/1773469195-image.png)

### 2\. 编写 httproute.yaml 并应用

![](/wp-uploads/2026/03/1773469239-image.png)

### 3\. 验证并删除 ingress web

![](/wp-uploads/2026/03/1773469270-image.png)

* * *

### 补充内容

#### Ingress，Gateway 和 HTTPRoute

Gateway 和 HTTPRoute 是 Kubernetes `Gateway API` 的两个核心资源，共同替代传统的 Ingress 实现更灵活的流量管理。

*   **Gateway 负责基础设施层**，定义`监听器`（端口、协议、主机名）和 `TLS 配置`，由平台团队管理；
*   **HTTPRoute 负责应用层**，定义`路由规则`（路径匹配、权重分发、后端服务），由应用团队管理。两者通过 `parentRefs` 关联，实现职责分离。

![](/wp-uploads/2026/03/1773469306-image.png)

Gateway 和 HTTPRoute

![](/wp-uploads/2026/03/1773469873-image.png)

Ingress

关系

说明

`Ingress`

第一代流量入口资源（已成熟，但功能有限）

`Gateway`

第二代流量入口的**基础设施层**（监听器+TLS）

`HTTPRoute`

第二代流量入口的**应用层**（路由规则）

`Gateway + HTTPRoute`

**替代 Ingress** 的新标准

#### 配置对比

**Ingress 配置（全部在一起）**:

```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  # ⚠️ TLS 配置在这里
  tls:
  - hosts:
    - example.com
    secretName: tls-secret

  # ⚠️ 路由规则也在这里
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Gateway + HTTPRoute 配置（职责分离）**:

```

## ===== Gateway（基础设施团队管理）=====
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: example.com
    # ⚠️ TLS 配置在这里
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: tls-secret

---
## ===== HTTPRoute（应用团队管理）=====
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  # ⚠️ 引用 Gateway
  parentRefs:
  - name: web-gateway
    kind: Gateway

  hostnames:
  - example.com

  # ⚠️ 路由规则在这里
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      kind: Service
      port:
        number: 80
```

#### 功能对比

功能

Ingress

Gateway + HTTPRoute

**HTTP 路由**

✅ 支持

✅ 支持

**HTTPS/TLS**

✅ 支持

✅ 支持（更灵活）

**路径匹配**

✅ 前缀/精确

✅ 前缀/精确/正则

**主机名匹配**

✅ 支持

✅ 支持

**多监听器**

❌ 有限

✅ 原生支持

**TCP/UDP**

❌ 不支持

✅ 支持 (TCPRoute/UDPRoute)

**gRPC**

❌ 不支持

✅ 支持 (GRPCRoute)

**流量拆分**

❌ 困难

✅ 原生支持

**跨命名空间**

❌ 困难

✅ 原生支持

**角色分离**

❌ 混合

✅ 清晰分离

**API 成熟度**

✅ GA

✅ GA (v1)

* * *