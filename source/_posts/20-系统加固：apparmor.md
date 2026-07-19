---
title: 20-系统加固：Apparmor
tags: []
id: '434'
categories:
  - - k8s 实验
date: 2026-07-16 13:59:46
---

##### 心智模型

最小权限原则，要求每个程序或系统进程这你能访问其任务所必须的信息和资源。

##### 实验：部署和引用 AppArmor Profile

##### 第一阶段：在 Worker 节点准备 Profile

AppArmor 是 Linux 内核模块，因此 Profile 必须加载到**每一个**运行 Pod 的节点内核中。

1.  **登录到你的 Worker 节点**。
2.  **创建 Profile 文件**：创建一个名为 `k8s-deny-write` 的配置文件，禁止容器写入任何文件。

Bash

```

cat <<EOF > /etc/apparmor.d/k8s-deny-write
profile k8s-deny-write flags=(attach_disconnected) {
  # 包含基本抽象
  include <abstractions/base>

  # 拒绝所有写入操作
  deny /** w,
}
EOF
```

1.  **加载 Profile 到内核**： 使用 `apparmor_parser` 命令解析并加载该配置文件。

Bash

```

sudo apparmor_parser -r -W /etc/apparmor.d/k8s-deny-write
```

1.  **确认加载成功**：

Bash

```

sudo aa-status  grep k8s-deny-write
```

* * *

第二阶段：在 Pod 中通过 Annotation 引用

在 Kubernetes 1.30 之前的版本中，AppArmor 主要通过 **Annotation (注解)** 进行配置。

1.  **编写 Pod 部署文件**： 注意 Annotation 的格式：`container.apparmor.security.beta.kubernetes.io/<container_name>: localhost/<profile_name>`。

YAML

```

apiVersion: v1
kind: Pod
metadata:
  name: apparmor-test-pod
  annotations:
    # 这里的 'test-container' 必须匹配下方的容器名
    # 'localhost/k8s-deny-write' 表示引用节点本地加载的 profile
    container.apparmor.security.beta.kubernetes.io/test-container: localhost/k8s-deny-write
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["sh", "-c", "echo 'Hello AppArmor' && sleep 3600"]
```

1.  **部署 Pod**：

Bash

```

kubectl apply -f apparmor-pod.yaml
```

* * *

##### 第三阶段：验证防御效果

我们要验证“最小权限”是否生效：

1.  **尝试在容器内创建文件**：

Bash

```

kubectl exec apparmor-test-pod -- touch /tmp/test.txt
```

1.  **预期结果**： 系统应报错：`touch: /tmp/test.txt: Permission denied`。
2.  **查看内核日志**（在 Worker 节点上）：

Bash

```

dmesg  grep -i apparmor
```

你会看到类似 `apparmor="DENIED"` 的记录，这证明内核成功拦截了违规操作。

##### 💡 关键要点总结

**步骤**

**核心操作**

**注意事项**

**节点侧**

加载 Profile 到内核

必须在**所有**潜在的 Worker 节点上执行。

**集群侧**

Pod Annotation 引用

格式必须极其精确，否则 Pod 可能无法启动或静默失败。

**维护**

审计与调优

生产环境通常先使用 `complain` 模式（仅记录不拦截）观察业务需求，再转为 `enforce`。

> **注意：** 从 Kubernetes 1.30 开始，AppArmor 已进入正式版 (GA)，推荐开始使用 `securityContext.appArmorProfile` 字段替代 Annotation，但目前社区中 Annotation 仍是最通用的学习和迁移方式。