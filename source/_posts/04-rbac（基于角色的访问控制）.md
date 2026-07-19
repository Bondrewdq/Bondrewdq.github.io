---
title: 04-RBAC（基于角色的访问控制）
tags: []
id: '217'
categories:
  - - k8s 实验
date: 2026-03-06 03:54:13
---

RBAC 是 Kubernetes 安全性的基石。考试中这类题目的逻辑非常固定，通常是：**“创建一个用户/账号 -> 赋予他某些权限 -> 验证他是否真的有了这些权限”**。

##### 实验目标

创建一个名为 `pod-reader-sa` 的服务账号（ServiceAccount），只允许它在 `development` 命名空间内**查看（get/list/watch）** Pod，禁止它进行删除或对其他资源的操作。

##### 实验过程大纲

阶段一：创建 namespace和 ServiceAccount

阶段二：创建 Role 和 RoleBinding

阶段三：验证权限

##### 实验过程

##### 阶段一：创建基础环境

首先，我们需要一个命名空间和这个账号。

```bash
## 1. 创建命名空间
kubectl create namespace development

## 2. 在该命名空间创建 ServiceAccount
kubectl create serviceaccount pod-reader-sa -n development
```

* * *

#### 阶段二：创建角色与绑定（核心操作）

在 RBAC 中，`Role` 定义“能做什么”，`RoleBinding` 定义“谁能做”。

**1\. 创建角色 (Role)** 我们需要定义一个角色，明确它只能对 Pod 执行读取操作。

```bash
## 使用命令行快捷创建（考试推荐用法，速度快且不易出错）
kubectl create role pod-reader \
  --namespace=development \
  --verb=get,list,watch \
  --resource=pods
```

**2\. 创建角色绑定 (RoleBinding)** 将刚才的 `ServiceAccount` 和 `Role` 关联起来。

```bash
kubectl create rolebinding read-pods-binding \
  --namespace=development \
  --role=pod-reader \
  --serviceaccount=development:pod-reader-sa
```

在 RBAC 的逻辑里，权限是可以“跨空间”授予的。比如可以给 A 空间账号拥有 B 空间权限。所以 --serviceaccount 这个选项的参数需要指定命名空间，不能直接只填 ServiceAccount。

* * *

#### 阶段三：验证权限

这不需要真的切换账号去操作，直接用 `kubectl auth can-i` 就能模拟该账号进行权限检查。

**1\. 验证“允许”的操作：** 检查 `pod-reader-sa` 是否能在 `development` 下查看 Pod：

```bash
kubectl auth can-i list pods --as=system:serviceaccount:development:pod-reader-sa -n development
```

> **预期输出：** `yes`

**2\. 验证“禁止”的操作（越权检查）：** 检查它是否能删除 Pod：

```

kubectl auth can-i delete pods --as=system:serviceaccount:development:pod-reader-sa -n development
```

> **预期输出：** `no`

**3\. 验证“跨空间”检查：** 检查它是否能查看 `default` 命名空间的 Pod（Role 是 Namespace 级别的，不应跨空间）：

```bash
kubectl auth can-i list pods --as=system:serviceaccount:development:pod-reader-sa -n default
```

> **预期输出：** `no`

##### 清理

```bash
kubectl delete ns development
```

#### 知识总结

**ServiceAccount 的全名格式**：

*   在使用 `--as` 模拟身份时，SA 的全称是 `system:serviceaccount:<namespace>:<serviceaccount-name>`。写错这个格式，验证就会失败。

**Verbs (动词)**：

*   常用的有：`get`, `list`, `watch`（读权限）；`create`, `update`, `patch`, `delete`（写权限）；`*`（所有权限）。

### 跨空间与集群级资源授权

#### **实验目标**

创建一个名为 `cluster-manager-sa` 的服务账号，赋予它：

1.  **集群级权限**：可以查看（get/list）集群中所有的 **Nodes**（节点）。
2.  **全空间权限**：可以查看 **所有命名空间** 下的 **Deployments**。

#### 实验过程大纲

1.  创建服务账号
2.  创建 ClusterRole 和 ClusterRoleBinding
3.  验证权限

#### 实验过程

##### 阶段一：创建服务账号

我们统一把这个账号放在 `kube-system` 命名空间下（模拟系统级管理账号）。

```bash
kubectl create serviceaccount cluster-manager-sa -n kube-system
```

* * *

##### 阶段二：创建 ClusterRole 与 ClusterRoleBinding

注意：这里**不需要**指定 `-n` 或 `--namespace`，因为 ClusterRole 是全局生效的。

**1\. 创建集群角色 (ClusterRole)** 定义对节点和所有 Deployment 的读取权限。

```

kubectl create clusterrole node-and-deploy-reader \
  --verb=get,list,watch \
  --resource=nodes,deployments
```

**2\. 创建集群角色绑定 (ClusterRoleBinding)** 将全局角色绑定到刚才的 ServiceAccount 上。

```

kubectl create clusterrolebinding cluster-manager-binding \
  --clusterrole=node-and-deploy-reader \
  --serviceaccount=kube-system:cluster-manager-sa
```

* * *

##### 阶段三：验证权限（多维度检查）

**1\. 验证集群资源（Nodes）：**

```

kubectl auth can-i list nodes --as=system:serviceaccount:kube-system:cluster-manager-sa
```

> **预期输出：** `yes`

**2\. 验证跨命名空间资源（Deployments）：** 检查它是否能看 `default` 空间的 Deployment：

```

kubectl auth can-i list deployments --as=system:serviceaccount:kube-system:cluster-manager-sa -n default
```

> **预期输出：** `yes`

检查它是否能看 `kube-system` 空间的 Deployment：

```

kubectl auth can-i list deployments --as=system:serviceaccount:kube-system:cluster-manager-sa -n kube-system
```

> **预期输出：** `yes`

**3\. 越权检查（验证安全性）：** 检查它是否能**删除**节点（我们只给了 get/list）：

```

kubectl auth can-i delete nodes --as=system:serviceaccount:kube-system:cluster-manager-sa
```

> **预期输出：** `no`

**清除实验痕迹：**

```bash
kubectl delete clusterrolebinding cluster-manager-binding
kubectl delete clusterrole node-and-deploy-reader
kubectl delete serviceaccount cluster-manager-sa -n kube-system
```

##### 知识总结

有时候考试会考一些冷门的资源，你不知道它的英文全称或是否支持 ClusterRole。这时候执行这条命令：

```

kubectl api-resources
```

*   **NAME 列**：填入 `--resource=` 的准确名称。
*   **NAMESPACED 列**：如果显示 `false`，则**必须**使用 ClusterRole。

1.  资源的作用域：Namespaced vs Non-Namespaced

*   **知识点：** Kubernetes 的资源分为两种。一种是躲在命名空间里的（如 Pods, Deployments, Services），另一种是漂浮在命名空间之上的集群级资源（如 **Nodes**, **PersistentVolumes**, **Namespaces**）。

2.  动词（Verbs）与资源（Resources）的精准匹配

*   **知识点：** RBAC 的核心三要素：**Who** (Subject), **What** (Verbs), **Which** (Resources)。
*   **避坑指南：**
    *   **复数形式：** 在 YAML 或命令行中，资源必须用复数，如 `nodes` 而不是 `node`。
    *   **子资源（Subresources）：** 考试有时会考 `deployment/status` 或 `pod/log`，这种斜杠写法代表对资源的特定部分进行授权。
    *   **API 组：** 有些资源属于不同的 API Group（如 Deployment 属于 `apps` 组）。使用命令 `kubectl create clusterrole --resource=deployments.apps` 可以确保精准匹配。