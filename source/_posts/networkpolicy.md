---
title: NetworkPolicy
tags: []
id: '330'
categories:
  - - k8s 实验
date: 2026-03-14 07:03:39
---

**NetworkPolicy = Kubernetes 里 Pod 级别的网络防火墙**，它基于标签选择器定义**允许**的入站（Ingress）和出站（Egress）流量，默认遵循“白名单”原则：一旦某个 Pod 被任何 NetworkPolicy 选中，未明确允许的流量都会被拒绝。

* * *

**实验要求**：

*   从提供的 YAML 样本中查看并应用适当的 NetworkPolicy。
*   确保选择的 NetworkPolicy **不过于宽松**，同时允许运行在 `frontend` 和 `backend` namespaces 中的 `frontend` 和 `backend` Deployment 之间的通信。
    *   首先，分析 `frontend` 和 `backend` Deployment，以确定需要应用的 NetworkPolicy 的具体要求。
    *   接下来，检查位于 `~/netpo`l 文件夹中的 NetworkPolicy YAML 示例。
        *   _注意：请勿删除或修改提供的示例。仅应用其中一个。否则可能会导致分数降低。_
    *   最后，应用启用 frontend 和 backend Deployment 之间的通信的 NetworkPolicy，但不要过于宽容。
        *   _注意：请勿删除或修改现有的默认拒绝所有入站流量或出口流量 NetworkPolicy。否则可能导致零分。_

参考链接：无  
难度：⭐⭐

* * *

### 0\. 环境确认

题干要求：“不过于宽松”，即最小权限原则

![](/wp-uploads/2026/03/1773471099-image.png)

查看\`~/netpol\`目录，发现存在 3 个文件，内容如下：

#### netpol1.yaml ❌ 过于宽松

```

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-1
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
```

**第 7 行，⚠️ 会选择 backend 命名空间 ALL Pods**

* * *

#### netpol2.yaml ✅ 正确选择

```

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-2
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend
```

*   **第 7 行，只选择 backend Pods**
*   **第 19 行，同时检查 namespace + Pod 标签**

* * *

#### netpol3.yaml ❌ 配置错误

```

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-3
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend
    - ipBlock:
        cidr: 10.0.0.0/24
```

*   **第 9 行，错误标签（应该是 backend）**
*   **第 21 行，⚠️ 额外开放整个网段，过于宽松**

* * *

### **1\. 选择 netpol2.yaml 并应用**

![](/wp-uploads/2026/03/1773471298-image.png)

### 2\. 检查

![](/wp-uploads/2026/03/1773471312-image.png)

_`netpol2.yaml` 中配置的 `NetworkPolicy` 位于命名空间 `backend`_

* * *