---
title: PriorityClass
tags: []
id: '308'
categories:
  - - k8s 实验
date: 2026-03-14 04:14:53
---

`PriorityClass` 是 Kubernetes 中用于定义 Pod 调度优先级的集群级资源，它通过数值（`value`）表示优先级高低，**数值越大优先级越高**。当集群资源不足时，调度器会优先调度高优先级的 Pod，并可能**驱逐**低优先级的 Pod 以腾出资源，从而确保关键业务在资源紧张时仍能正常运行。

* * *

**实验要求**：

*   请执行以下任务：
    *   为用户工作负载创建一个名为 `high-priority` 的新 PriorityClass ，其值比用户定义的现有最高优先级类值小一。
    *   修改在 `priority` namespace 中运行的现有 `busybox-logger` Deployment ，以使用 `high-priority` 优先级类。
    *   确保 `busybox-logger` Deployment 在设置了新优先级类后成功部署。
*   _请勿修改在 `priority` namespace 中运行的其他 Deployment，否则可能导致分数降低。_

参考链接： [https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)  
难度：⭐⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773461404-image.png)

### 1\. 查找用户自定义 priorityClass（pc）

```

kubectl get priorityclass
```

其中 `system-cluster-critical` 和 `system-node-critical` 是集群默认带的，而最有一个 `max-user-priority` 是用户自定义的。其值为 1000000000（十位数），所以小一就是九个 9（999999999）

### 2 .创建 pc.yaml 并应用

![](/wp-uploads/2026/03/1773461488-image.png)

### 3\. 修改 deployment

```

kubectl edit deployment busybox-logger -n priority
```

在 `spec.template.spec` 中添加：`priorityClassName: high-priority`

![](/wp-uploads/2026/03/1773461570-image.png)

### 4\. 验证

![](/wp-uploads/2026/03/1773461591-image-1024x308.png)

_真正考试时，一开始这个 Pod 是非 Running 的异常状态。等修改完上一步后，大约等 2 分钟，新生成的 Pod busybox-logger 会是 Running 状态。_

* * *

### 补充内容

#### Pod 优先级和抢占

Pod 可以有**优先级**。 优先级表示一个 Pod 相对于其他 Pod 的重要性。 如果一个 Pod 无法被调度，调度程序会尝试**抢占**（**驱逐**）较低优先级的 Pod， 以使悬决 Pod 可以被调度。

特性

**非抢占式**

**抢占式**

**preemptionPolicy**

`Never`

`PreemptLowerPriority` (默认)

**行为**

资源不足时**等待**

资源不足时**驱逐低优先级 Pod**

**安全性**

✅ 高（不影响其他 Pod）

⚠️ 低（可能影响其他 Pod）

**适用场景**

重要但不紧急的任务

关键任务（必须立即运行）

```

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never        # 抢占式配置：PreemptLowerPriority
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

* * *