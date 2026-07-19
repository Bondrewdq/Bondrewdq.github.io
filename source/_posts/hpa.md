---
title: HPA
tags: []
id: '264'
categories:
  - - k8s 实验
date: 2026-03-12 11:56:53
---

`HPA`（`水平 Pod 自动扩缩容`，`Horizontal Pod Autoscaler`） 是 Kubernetes 中根据 **CPU/内存使用率** 或 **自定义指标** 自动调整 Pod 副本数量的机制。

* * *

**实验要求**：

*   在 `autoscale namespace` 中创建一个名为 `apache-server` 的新 `HorizontalPodAutoscaler(HPA)`。此 `HPA` 必须定位到 `autoscale namespace` 中名为 `apache-server` 的现有 `Deployment` 。
*   将 `HPA` 设置为每个 `Pod` 的 CPU 使用率旨在 50% 。将其配置为至少有 1 个 `Pod`，且不超过 4 个 `Pod` 。此外，将缩小稳定窗口设置为 `30` 秒。

参考链接： [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

参考难度：⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773314478-image.png)

* * *

### 1.1 方法一：命令行快速创建（推荐）

```

## 1. 创建 HPA（基本配置）
kubectl autoscale deployment apache-server \
  --name=apache-server \
  --namespace=autoscale \
  --min=1 \
  --max=4 \
  --cpu=50%  # 这是 1.34 的写法

## 2. 编辑 HPA 添加缩小稳定窗口
kubectl edit hpa apache-server -n autoscale

## 3. 在 spec 中添加 behavior 配置
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```

![](/wp-uploads/2026/03/1773314591-image-1024x43.png)

![](/wp-uploads/2026/03/1773314598-image.png)

![](/wp-uploads/2026/03/1773314608-image-1024x59.png)

#### 注意事项

*   `spec.behavior.scaleDown.stabilizationWindowSecends`
*   不要漏了带上命名空间 `-n sutoscale`

* * *

### 1.2 方法二：YAML 文件创建

```

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-server
  namespace: autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-server
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30  # ⚠️ 缩小稳定窗口 30 秒
    scaleUp:
      stabilizationWindowSeconds: 0   # 扩容立即生效（可选）
```

* * *

### 补充内容

`behavior` 字段允许精细控制扩缩容的**速度**和**稳定性**，防止因指标波动导致服务不稳定

特性

**scaleUp (扩容)**

**scaleDown (缩容)**

**目标**

🚀 **快速响应**负载激增

🐢 **谨慎稳定**防止抖动

**默认策略**

立即扩容 (0 秒稳定窗口)

等待 300 秒 (5 分钟)

**风险**

资源不足导致服务不可用

频繁启停导致服务震荡

**配置倾向**

激进

保守

#### 1️⃣ scaleUp (扩容行为)

1.  **作用**：当负载**超过**目标阈值时，控制 Pod 增加的速度。
2.  **典型场景**：流量突然暴涨（如秒杀活动），需要**立即**增加 Pod 以承载压力
3.  **配置示例**：

```

behavior:
  scaleUp:
    stabilizationWindowSeconds: 0    # ⚡ 0 秒：检测到负载高立即扩容
    policies:
    - type: Percent                  # 策略 1：按百分比
      value: 100                     # 每次最多扩容 100%
      periodSeconds: 15              # 每 15 秒评估一次
    - type: Pods                     # 策略 2：按数量
      value: 4                       # 每次最多增加 4 个 Pod
      periodSeconds: 15
    selectPolicy: Max                # 取两者中扩容更多的策略
```

4.  **关键参数**：

参数

说明

推荐值

`stabilizationWindowSeconds`

稳定窗口（秒）

`0` (立即响应)

`policies`

扩容速率限制

防止瞬间创建过多 Pod

`selectPolicy`

多策略选择

`Max` (最快) / `Min` / `Disabled`

* * *

#### 2️⃣ scaleDown (缩容行为)

1.  **作用**：当负载**低于**目标阈值时，控制 Pod 减少的速度。
2.  **典型场景**：流量高峰过去，负载下降。如果立即缩容，万一流量再次回升，会导致服务反复启停（震荡）。
3.  **配置示例**：

```

behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # ⏳ 300 秒：持续 5 分钟低负载才缩容
    policies:
    - type: Percent
      value: 10                      # 每次最多缩容 10%
      periodSeconds: 60
    selectPolicy: Min                # 取最保守的策略（最慢）
```

4.  **关键参数**：同上表

* * *