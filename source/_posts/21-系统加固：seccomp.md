---
title: 21-系统加固：Seccomp
tags: []
id: '436'
categories:
  - - k8s 实验
date: 2026-07-16 14:00:35
---

**Seccomp (Secure Computing mode)** 是 Linux 内核的一项功能，用于限制容器可以发起的**系统调用 (System Calls)**。

##### 🛠️ 实操手册：配置 Seccomp Profile

在 Kubernetes 中，Seccomp 已经比 AppArmor 更加标准化，直接集成在 `securityContext` 字段中。

##### 第一阶段：使用 `RuntimeDefault` (最推荐的生产配置)

这是容器运行时（如 containerd 或 Docker）自带的一套“安全模板”，它禁用了大约 40 多个危险的系统调用。

1.  **编写 Pod 配置文件**： 与 AppArmor 不同，Seccomp 建议直接写在 `spec` 字段中。

YAML

```

apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default-pod
spec:
  securityContext:
    # 在 Pod 级别应用 seccomp
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: web-app
    image: nginx
```

* * *

##### 第二阶段：自定义 Seccomp Profile (精细化控制)

有时候默认配置还是太宽松，或者你需要运行极其特殊的应用。

1.  **在 Worker 节点创建 Profile 目录**： Kubelet 默认在特定的安全目录下寻找 Profile。
    
    Bash
    
    ```
    # 在所有 Worker 节点执行
    sudo mkdir -p /var/lib/kubelet/seccomp/profiles
    ```
    
2.  **编写 JSON Profile**： 创建一个名为 `audit-only.json` 的文件，它的作用是：允许大部分操作，但对某些操作记录日志（用于审计）。
    

JSON

```

{
    "defaultAction": "SCMP_ACT_ALLOW",
    "architectures": [
        "SCMP_ARCH_X86_64"
    ],
    "syscalls": [
        {
            "names": [
                "reboot"
            ],
            "action": "SCMP_ACT_KILL"
        }
    ]
}
```

> **语法解释**：如果进程尝试调用 `reboot`（重启），内核会直接杀掉 (`SCMP_ACT_KILL`) 该进程。

1.  **将文件放入 Worker 节点指定路径**： 将此 JSON 放在 `/var/lib/kubelet/seccomp/profiles/audit-only.json`。
2.  **在 Pod 中引用自定义 Profile**：

YAML

```

apiVersion: v1
kind: Pod
metadata:
  name: seccomp-custom-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      seccompProfile:
        type: Localhost
        # 注意：这里的路径是相对于 /var/lib/kubelet/seccomp/ 的相对路径
        localhostProfile: profiles/audit-only.json
```

* * *

##### 第三阶段：验证实验结果

1.  **部署 Pod**：
    
    Bash
    
    ```
    kubectl apply -f seccomp-custom-pod.yaml
    ```
    
2.  **测试拦截效果**： 由于我们设置了 `reboot` 就会被 `KILL`，我们可以进入容器尝试触发这个调用。
    
    Bash
    
    ```
    kubectl exec seccomp-custom-pod -- reboot
    ```
    
3.  **预期现象**： 你会发现连接被断开，或者提示 `command terminated with exit code 137`。这是因为内核检测到非法调用，瞬间终止了进程。
    

##### 避坑指南

*   **路径陷阱**：在 Pod YAML 中写 `localhostProfile` 时，**不要**写绝对路径 `/var/lib/...`。Kubelet 会自动拼凑前缀。
*   **权限陷阱**：由于 Seccomp 需要在每个节点部署 JSON 文件，建议在生产中使用 **DaemonSet** 来自动把 Profile 文件同步到所有节点的磁盘上。