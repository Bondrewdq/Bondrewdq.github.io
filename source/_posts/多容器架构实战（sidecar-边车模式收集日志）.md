---
title: 16-多容器架构实战（Sidecar 边车模式收集日志）
tags: []
id: '419'
categories:
  - - k8s 实验
date: 2026-07-16 13:35:25
---

在真实的生产环境中，很多老旧系统（Legacy Systems）并不符合云原生标准。它们不会把日志输出到终端（`stdout`），而是死板地写进容器内的某个文件里（比如 `/var/log/app.log`）。这就导致 Kubernetes 原生的 `kubectl logs` 命令直接失效（因为它只抓取容器的 `stdout`）。

#### 核心技术逻辑：Sidecar（边车）模式与共享卷

要在不修改老旧代码的前提下解决这个问题，K8s 给出的标准答案是：**Sidecar 模式**。

1.  **Pod 的本质：** Pod 不是一个容器，而是一个“容器组”。同一个 Pod 内的多个容器共享网络（localhost）和存储卷。
2.  **EmptyDir 卷（桥梁）：** 我们创建一个生命周期与 Pod 绑定的临时存储卷 `emptyDir`，把它同时挂载给这两个容器。
3.  **数据流向：**
    *   **业务容器 (Main)：** 把日志不断写入挂载的目录中（例如 `/var/log/app/sys.log`）。
    *   **边车容器 (Sidecar)：** 同样挂载这个目录，只运行一个极为简单的命令（如 `tail -f /var/log/app/sys.log`），把文件内容实时读取出来，并输出到**自己的**终端（`stdout`）。
4.  **最终结果：** 此时，你只需要用 `kubectl logs` 去看那个 Sidecar 容器，就能完美获取老旧系统的日志了！

* * *

#### 实战挑战

我们将从零手写一个包含两个容器的 Pod YAML。

**1\. 编写多容器 Pod 配置文件 `sidecar-pod.yaml`：**

```

apiVersion: v1
kind: Pod
metadata:
  name: legacy-app-pod
spec:
  # 1. 声明一个共享的空目录卷
  volumes:
  - name: shared-logs
    emptyDir: {}

  containers:
  # 2. 这是老旧的业务主容器（它只往文件里写，不往屏幕上打）
  - name: main-app
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo '$(date) - ERROR: legacy system failure' >> /var/log/app/sys.log; sleep 2; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app   # 挂载到业务容器的这个路径

  # 3. 这是我们注入的 Sidecar 日志收集容器
  - name: log-sidecar
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /var/log/app/sys.log"] # 实时读取文件并输出到标准输出
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app   # 必须挂载同一个卷到相同的（或不同的）路径
```

**2\. 部署这个“双黄蛋” Pod：**

```

kubectl apply -f sidecar-pod.yaml
```

**3\. 观察多容器的就绪状态：**

```

kubectl get pod legacy-app-pod -w
```

> **预期现象：** 注意看 `READY` 那一列。普通的 Pod 是 `1/1`，而这个 Pod 在启动时会显示 `0/2`，最终变成 `2/2`。这代表里面的两个容器都已成功运行。

* * *

#### 阶段二：见证 Sidecar 的魔法 (CKA 必考命令)

现在，我们来验证日志提取。

**1\. 尝试直接获取 Pod 日志（会报错）：** 在单容器时代，你习惯直接敲 `kubectl logs <pod-name>`。但在多容器时代，这行不通了：

```

kubectl logs legacy-app-pod
```

> **预期输出：** `error: a container name must be specified for pod legacy-app-pod...` K8s 会抱怨：你这 Pod 里有两个容器，你到底要看谁的？

**2\. 明确指定容器名称（查看主容器）：** 使用 `-c` 参数指定容器名。

```

kubectl logs legacy-app-pod -c main-app
```

> **预期输出：** 空空如也。因为老旧系统没有向终端输出任何东西。

**3\. 见证奇迹时刻（查看 Sidecar 容器）：**

```

kubectl logs legacy-app-pod -c log-sidecar
```

> **预期输出：** 你会看到屏幕上源源不断地打印出带有时间戳的 `ERROR: legacy system failure`！

这就是 Sidecar 模式的魅力：**解耦**。业务容器专心跑业务，日志容器专心搬运日志，互不干扰，却完美协同。

* * *

#### 清理

```

kubectl delete pod legacy-app-pod
```