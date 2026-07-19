---
title: Service
tags: []
id: '303'
categories:
  - - k8s 实验
date: 2026-03-14 04:03:28
---

*   `svc` 为一组 Pod 提供稳定的**网络访问入口**，通过标签选择器 `Label Selector` 动态关联后端 Pod，**屏蔽其频繁变化的 IP**。实现了**服务发现**与**负载均衡**，无需关心底层实例的生命周期。

* * *

**实验要求**：

*   重新配置 `spline-reticulator` `namespace` 中现有的 `front-end` Deployment，以公开现有容器 nginx 的端口 80/tcp
*   创建一个名为 `front-end-svc` 的新 Service ，以公开容器端口 80/tcp
*   配置新的 Service ，以通过 `NodePort` 公开各个 Pod

参考链接：无  
难度： ⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773460766-image.png)

### 1\. 方法一：直接修改 deployment（推荐）

1.  修改 deployment：`kubectl edit deployment -n spline-reticulator front-end`  
    **使容器开放80端口（TargetPort）**

![](/wp-uploads/2026/03/1773460798-image.png)

2.  为 deployment 维护的一系列 Pod **手动暴露** svc：

```

kubectl expose deployment -n spline-reticulator front-end \
  --type=NodePort --port=80 --target-port=80 --name=front-end-svc
```

### 2\. 方法二：创建 svc.yaml

```

apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
  namespace: spline-reticulator
spec:
  type: NodePort
  selector:
    app: front-end           # ⚠️ 需要匹配 Deployment 的 Pod 标签
  ports:
  - port: 80                 # Service 端口
    targetPort: 80           # 容器端口
    nodePort: 30080          # NodePort 端口 (30000-32767)
    protocol: TCP
```

### 3\. 验证

![](/wp-uploads/2026/03/1773460916-image.png)

* * *