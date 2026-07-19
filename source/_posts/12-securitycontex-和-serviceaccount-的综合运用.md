---
title: 12-SecurityContex 和 ServiceAccount 的综合运用
tags: []
id: '251'
categories:
  - - k8s 实验
date: 2026-03-11 09:08:55
---

#### 核心心智模型：API 边界 vs 宿主机内核边界

在一个运行的容器（Pod）中，它的“权限”被严格划分为两个完全不同的维度：

1.  **ServiceAccount (服务账号) —— 突破 Kubernetes API 边界**
    *   **技术本质：** 它是与 `kube-apiserver` 通信的凭证（JWT Token）。
    *   **作用域：** 决定了容器内的进程是否能调用 K8s API 去操作集群资源（比如：读取 Secret、删除 Pod、查看 Node）。它完全不关心容器在 Linux 系统层面的权限。
2.  **SecurityContext (安全上下文) —— 限制 Linux 宿主机内核边界**
    *   **技术本质：** 它是直接映射到底层 Linux 内核特性的配置（如 UID/GID 映射、Linux Capabilities、只读文件系统、特权模式 Privilege Escalation）。
    *   **作用域：** 决定了容器内的进程在宿主机操作系统层面的能力（比如：能否修改系统时间、能否绑定 80 端口、能否以 root 用户运行）。它完全不关心 K8s API。

#### 实验目标

构建一个“API 特权，但系统受限”的综合体

我们将创建一个测试 Namespace，在里面放一个极其重要的 Secret。 然后创建一个 Pod，要求它有权限通过 API 读取这个 Secret（利用 SA），但在 Linux 系统底层，它被剥夺了 root 权限，无法修改容器内的系统文件（利用 SC）。

#### 实验过程

##### 阶段一：准备 API 权限 (ServiceAccount & RBAC)

**1\. 创建隔离环境和测试数据**

```bash
kubectl create ns sec-test
## 创建一个机密数据
kubectl create secret generic top-secret --from-literal=password=CKA_PASS_2026 -n sec-test
```

**2\. 创建凭证 (ServiceAccount) 与授权** 这部分我们复习一下之前学过的 RBAC：

```bash
## 1. 创建账号
kubectl create serviceaccount api-reader -n sec-test

## 2. 创建角色（只允许读取 Secret）
kubectl create role secret-reader --verb=get,list --resource=secrets -n sec-test

## 3. 绑定权限
kubectl create rolebinding api-reader-bind --role=secret-reader --serviceaccount=sec-test:api-reader -n sec-test
```

* * *

##### 阶段二：结合安全上下文拉起工作负载

现在，我们编写一个综合了 `ServiceAccount` 和 `SecurityContext` 的 Pod YAML。

**1\. 创建文件 `secure-pod.yaml`** 仔细看注释，理解这两个字段在 YAML 结构中的位置（这也是 CKA 考点）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-test-pod
  namespace: sec-test
spec:
  serviceAccountName: api-reader # 1. API 边界：注入我们刚才创建的 API 凭证
  securityContext:               # 2. Pod 级 OS 边界：强制该 Pod 内所有容器以非 root 用户运行
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: test-container
    image: curlimages/curl       # 使用包含 curl 的基础镜像
    command: ["sleep", "3600"]
    securityContext:             # 3. 容器级 OS 边界：禁止提权，并把根文件系统设为只读
      allowPrivilegeEscalation: false# 禁止提权
      readOnlyRootFilesystem: true# 根文件系统只读
```

**2\. 部署 Pod**

```bash
kubectl apply -f secure-pod.yaml
kubectl get pods -n sec-test
```

* * *

##### 阶段三：硬核验证（进入容器内部探针）

现在，我们要 `exec` 到容器内部，分别验证这两个维度的权限配置是否生效。

```bash
kubectl exec -it security-test-pod -n sec-test -- sh
```

_(以下命令均在容器内部的 `sh` 终端执行)_

**验证一：OS 边界拦截 (SecurityContext 生效)** 尝试在 Linux 系统层面执行越权操作：

```bash
## 1. 查看当前用户身份
id
## 预期输出：uid=1000 gid=3000... (证明 runAsUser 生效，你不再是 root)

## 2. 尝试在根目录创建一个文件
touch /test.txt
## 预期输出：Read-only file system (证明 readOnlyRootFilesystem 生效，即便你是 root 也写不进去)
```

**验证二：API 边界突破 (ServiceAccount 生效)** 容器内部并没有安装 `kubectl`，我们需要手动构造 HTTP 请求，利用注入到容器里的 Token 直接调用 Kube-apiserver。这能让你彻底看清 ServiceAccount 的底层工作原理。

```bash
## 1. 将 K8s 自动注入到容器的 Token 和 CA 证书路径赋值给变量
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

## 2. 带着 Token 发起 HTTPS 请求，调用 K8s API 去读取那个 Secret
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/sec-test/secrets/top-secret
```

> **预期现象：** 虽然你在 Linux 系统里是个连文件都写不了的“废人”（UID 1000），但由于你持有 `api-reader` 的 Token，Kube-apiserver 会给你返回一长串 JSON 数据，里面包含了 Base64 编码后的机密密码！

_(验证完毕后，输入 `exit` 退出容器)_

* * *

#### CKA 考点提炼与技术总结

1.  **YAML 结构层级：**
    *   `spec.securityContext`：作用于 Pod 内的**所有容器**（如 `runAsUser`, `fsGroup`）。
    *   `spec.containers[].securityContext`：仅作用于**当前特定容器**（如 `readOnlyRootFilesystem`, `privileged: true`）。
    *   `spec.serviceAccountName`：属于 Pod 级别，一个 Pod 只能拥有一个 API 身份。
2.  **默认挂载行为：** 只要你指定了 `serviceAccountName`，kubelet 就会自动把对应的 Token、CA 证书和 Namespace 信息以文件的形式挂载到容器的 `/var/run/secrets/kubernetes.io/serviceaccount/` 目录下。
3.  **权限最小化原则：** 在生产环境中，对于暴露对外的 Web 服务，标准做法就是结合两者：**禁止以 root 运行（SC），且使用默认或受限的 SA（防 API 越权）**。

#### 清理实验战场

```bash
kubectl delete ns sec-test
```