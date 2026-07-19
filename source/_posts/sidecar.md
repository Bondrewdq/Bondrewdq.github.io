---
title: Sidecar
tags: []
id: '291'
categories:
  - - k8s 实验
date: 2026-03-14 03:32:25
---

Sidecar 模式是 **Kubernetes（K8s）中最常用的容器设计模式之一**，核心是在一个 **Pod 内** 与**主业务容器**并行运行一个**辅助容器（Sidecar 容器）**，两者共享 Pod 的网络、存储、PID 等命名空间，辅助主容器完成非业务核心的支撑性工作，且完全解耦主业务逻辑。

* * *

**实验要求：**

*   Context
    *   您需要将一个传统应用程序集成到 Kubernetes 的日志架构(例如 kubectl logs)中。
    *   实现这个要求的通常方法是添加一个流式传输并置容器。
*   Task
    *   更新现有的 synergy-leverager Deployment，
    *   将使用 busybox:stable 镜像，且名为 sidecar 的并置容器，添加到现有的 Pod 。
    *   新的并置容器必须运行以下命令：
        *   /bin/sh -c "tail -n+1 -f /var/log/synergy-leverager.log"
*   使用挂载在 /var/log 的 Volume，使日志文件 synergy-leverager.log 可供并置容器使用。
*   除了添加所需的卷挂载之外，请勿修改现有容器的规范。

参考链接： [https://kubernetes.io/docs/concepts/cluster-administration/logging](https://kubernetes.io/docs/concepts/cluster-administration/logging)  
难度： ⭐⭐

* * *

### 0\. 查看环境

![](/wp-uploads/2026/03/1773456675-image.png)

![](/wp-uploads/2026/03/1773456702-image-1024x851.png)

### 1\. 修改 Deployment

直接编辑：`kubectl edit deploy synergy-leverager`

![](/wp-uploads/2026/03/1773456903-image.png)

### 3\. 验证

![](/wp-uploads/2026/03/1773456955-image.png)

### 4\. 陷阱

#### 4.1 原容器也要挂载日志卷

如果原有容器不挂载该卷，_它产生的日志_只会留在自己的文件系统中，Sidecar 容器挂载的 `emptyDir` 将会是空的，看不到任何日志

#### 4.2 volumes 应与 spec.template.spec.containers 同级

* * *

### 补充内容

#### 1️⃣ Sidecar 模式理解（必考）

![](/wp-uploads/2026/03/1773456435-image.png)

* * *

#### 2️⃣ 多容器 Pod 配置（核心考点）

```

spec:
  containers:
  # 容器 1：主应用
  - name: synergy-leverager
    image: <原有镜像>
    volumeMounts:
    - name: log-volume
      mountPath: /var/log      # ⚠️ 共享同一个 Volume

  # 容器 2：Sidecar
  - name: sidecar              # ⚠️ 容器名称必须指定
    image: busybox:stable
    command: ["/bin/sh", "-c"]
    args: ["tail -n+1 -f /var/log/synergy-leverager.log"]
    volumeMounts:
    - name: log-volume         # ⚠️ 共享同一个 Volume
      mountPath: /var/log

  # Volume 定义（Pod 级别）
  volumes:
  - name: log-volume           # ⚠️ Volume 名称
    emptyDir: {}               # 或 hostPath
```

考点

细节

容器名称

多容器 Pod 必须指定 `name`

Volume 名称

`volumeMounts.name` 必须等于 `volumes.name`

挂载路径

两个容器都挂载到 `/var/log`

命令格式

`command` + `args` 或 `command: ["/bin/sh", "-c", "命令"]`

* * *

#### 3️⃣ Volume 类型选择（隐含考点）

Volume 类型

适用场景

本题推荐

**emptyDir**

Pod 内容器共享临时数据

✅ 推荐

**hostPath**

需要持久化到节点

⚠️ 谨慎使用

**persistentVolumeClaim**

跨 Pod 持久化

❌ 不必要

```

## emptyDir（推荐）
volumes:
- name: varlog
  emptyDir: {}

## hostPath（如果题目要求）
volumes:
- name: varlog
  hostPath:
    path: /var/log
    type: DirectoryOrCreate
```

* * *