---
title: StorageClass
tags: []
id: '298'
categories:
  - - k8s 实验
date: 2026-03-14 03:44:04
---

存储类**StorageClass** ，是 Kubernetes 中**动态存储供给**的核心机制，它让 PVC 可以自动创建 PV，无需管理员手动预配置

* * *

**实验要求**：

*   首先，为名为 `rancher.io/local-path` 的现有制备器，创建一个名为 `ran-local-path` 的新 `StorageClass`
*   将卷绑定模式设置为 `WaitForFirstConsumer`
    *   _注意，没有设置卷绑定模式，或者将其设置为 `WaitForFirstConsumer` 之外的其他任何模式，都将导致分数降低。_
*   接下来，将 `ran-local-path` `StorageClass` 配置为默认的 `StorageClass`
    *   _请勿修改任何现有的 `Deployment` 和 `PersistentVolumeClaim`，否则将导致分数降低。_

参考链接： [https://kubernetes.io/docs/concepts/storage/storage-classes/](https://kubernetes.io/docs/concepts/storage/storage-classes/)  
难度： ⭐

* * *

### 0\. 查看环境

![](/wp-uploads/2026/03/1773459674-image-1024x312.png)

### 1\. 创建 sc.yaml

![](/wp-uploads/2026/03/1773459688-image.png)

```

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ran-local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # ⚠️ 设为默认
provisioner: rancher.io/local-path                       # ⚠️ 指定 provisioner
volumeBindingMode: WaitForFirstConsumer                  # ⚠️ 关键配置
reclaimPolicy: Delete                                    # 可选
allowVolumeExpansion: true                               # 可选
```

### 2\. 验证配置

![](/wp-uploads/2026/03/1773459753-image-1024x108.png)

* * *

### 补充内容

#### VolumeBindingMode

```

volumeBindingMode: WaitForFirstConsumer  # ⚠️ 题目明确要求
```

模式

说明

本题要求

`Immediate`

PVC 创建时立即绑定 PV

❌ 不得分

`WaitForFirstConsumer`

Pod 调度后再绑定 PV

✅ 必须

_`WaitForFirstConsumer` 确保 PV 创建在与 Pod 相同的节点上，对于本地存储（`local-path`）至关重要_

* * *