---
title: 18-集群设置：kube-bench
tags: []
id: '428'
categories:
  - - k8s 实验
date: 2026-07-16 13:54:29
---

`kube-bench` 进行集群安全加固是 Kubernetes 运维中的高级进阶任务。它主要基于 **CIS (Center for Internet Security) Kubernetes Benchmark** 标准来检查集群配置。

##### 心智模型：

在操作之前，需要建立这样一个逻辑回路：

1.  **扫描 (Scan)：** `kube-bench` 作为一个二进制工具或容器运行，它会读取主机的进程参数（如 `ps -ef grep kube-apiserver`）和配置文件。
2.  **比对 (Compare)：** 它将获取到的参数与 CIS 标准进行比对。
3.  **报告 (Report)：** 输出 `[PASS]`（通过）、`[FAIL]`（失败）或 `[WARN]`（警告）。每个 `[FAIL]` 都会附带一个 `Remediation`（修复建议）。
4.  **修复 (Fix)：** 修改 `/etc/kubernetes/manifests/` 下的 YAML。由于这些是 **Static Pod**，`kubelet` 会监控文件变化并自动重启组件应用新参数。

##### 实验步骤指南

##### 方式一（推荐）：直接用 Job

```

kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

查看结果：

```

kubectl logs job/kube-bench
```

* * *

你会看到类似：

```

[PASS] 1.1.1 Ensure API server pod specification file permissions are set to 644 or more restrictive
[FAIL] 1.2.7 Ensure that the --authorization-mode argument includes Node
```

**2.定位漏洞与修复**

假设报告中出现以下失败项：

*   **\[FAIL\] 1.2.7 Ensure that the --authorization-mode argument includes Node**
*   **\[FAIL\] 1.2.19 Ensure that the --profiling argument is set to false**

**3\. 修改 Manifest**

进入目录并编辑 API Server 的定义文件：

```

cd /etc/kubernetes/manifests/
vi kube-apiserver.yaml
```

在 `spec.containers.command` 列表下添加或修改参数：

```

- --authorization-mode=Node,RBAC  # 确保包含 Node
- --profiling=false               # 关闭分析接口以减少攻击面
```

保存退出后，等待约 30 秒，使用 `kubectl get pods -n kube-system` 查看 api-server 是否重启成功

##### Kubernetes 安全 = 4层防线

```

① 身份（RBAC / authn）
② 权限（authorization）
③ 数据（etcd encryption）
④ 节点（kubelet security）
```