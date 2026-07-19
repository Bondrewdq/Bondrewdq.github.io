---
title: PVC
tags: []
id: '314'
categories:
  - - k8s 实验
date: 2026-03-14 04:26:57
---

`PV (PersistentVolume)` 和 `PVC (PersistentVolumeClaim)` 是 Kubernetes 中**持久化存储**的核心机制，解决了 hostPath 的节点绑定问题 。

*   `PV` ：代表 K8s 中的存储资源（_允许用户将外部存储资源映射到集群_），由管理员（存储部门提供）
*   `PVC`：是用户对 PV 的**申请单**，赋予 Pod 访问 PV 的权限

![](/wp-uploads/2026/03/1773462012-image.png)

* * *

**实验要求**：

*   `mariadb` namespace 中的 `MariaDB` Deployment 被误删除。请恢复该 Deployment 并确保数据持久性。请按照以下步骤：
    *   如下规格在 `mariadb` namespace 中创建名为 `mariadb` 的 PersistentVolumeClaim (PVC)：
        *   访问模式为 `ReadWriteOnce`
            *   存储为 250Mi
    *   集群中现有一个 PersistentVolume。
    *   您必须使用现有的 PersistentVolume (PV)。
    *   编辑位于 `~/mariadb-deployment.yaml` 的 `MariaDB` Deployment 文件，以使用上一步中创建的 PVC。
    *   将更新的 Deployment 文件应用到集群。
    *   确保 `MariaDB` Deployment 正在运行且稳定。

参考链接： [https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)  
难度：⭐⭐

* * *

### 0\. 环境确认

![](/wp-uploads/2026/03/1773462183-image-1024x156.png)

注意 `mariadb下` PV 的\`StorageClass\`字段，后面 PVC 编写时需要

### 1\. 编写 pvc.yaml 并应用

![](/wp-uploads/2026/03/1773462284-image.png)

根据[参考链接](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)查询到的模板，修改字段即可

### 2\. 修改 mariadb-deployment.yaml 并应用

`vim maria-deployment.yaml`

![](/wp-uploads/2026/03/1773462381-image.png)

### 3\. 验证

![](/wp-uploads/2026/03/1773462396-image-1024x148.png)

* * *