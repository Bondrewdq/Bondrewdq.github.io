---
title: 17-使用 etcdctl 练习备份与恢复集群数据
tags: []
id: '424'
categories:
  - - k8s 实验
date: 2026-07-16 13:46:01
---

ectdctl 是一个命令行界面 （CLI）工具，专门用于与 etcd 数据库进行交互。

目前 ectd 普遍使用 v3 版本的 API ，操作前需要指定环境变量 `ETCDCTL_API=3`。

新版将 etcdctl 功能拆分了：

*   `ectdctl`:用于与正在运行的 etcd 服务交互 （通过网络）
*   `etcdutl`:用于处理离线文件

##### 第一部分：etcd 备份与恢复的心智模型

1.  集群所有状态都以键值对形式存储在 etcd 中。除了 kube-apiserver，没有任何组件可以直接和 etcd 对话。
2.  备份的本质是拷贝那一瞬间 etcd 底层 BoltDB 引擎的物理文件。
3.  恢复操作不是覆盖，而是那旧的快照文件，在一个全新的目录下解压重现当时的数据结构，然后修改 etcd 配置文件，指向这个新目录。

##### 第二部分：实战演练指南

##### 第一步：准备“万能钥匙”（设置 Alias）

在 kubeadm 搭建的集群中，etcd 启用了双向 TLS 认证。每次敲击 `etcdctl` 都需要带上一长串证书路径，非常反人类。我们先设置一个临时别名来简化操作：

```bash
## 确认 etcdctl 已安装 (如果没有，可通过 apt/yum 安装 etcd-client)
## 设置别名，指定 API 版本为 3，并带上 kubeadm 默认的 etcd 证书路径
alias etcdctl='ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'
```

_验证一下：_ 运行 `sudo etcdctl endpoint health`，如果返回 `is healthy`，说明钥匙管用。

##### 第二步：制造“案发现场”（创造测试数据）

我们在集群里留下一些独特的痕迹，证明当前的“时间线”。

```bash
kubectl create namespace disaster-recovery
kubectl run survivor-pod --image=nginx -n disaster-recovery
kubectl get pods -n disaster-recovery
## 确保 pod 处于 Running 状态
```

##### 第三步：时空冻结（执行备份）

现在，我们将当前的集群状态备份到 `/opt/etcd-backup.db` 文件中。

```bash
etcdctl snapshot save /opt/etcd-backup.db
```

_验证一下：_ 运行 `etcdutl snapshot status /opt/etcd-backup.db -w table`，你会看到备份文件的详细哈希和体积。

##### 第四步：模拟灾难（破坏数据）

现在，假装有新手误删了整个核心业务线。

```bash
kubectl delete namespace disaster-recovery
```

_验证一下：_ 此时再运行 `kubectl get pods -n disaster-recovery`，会提示 Namespace 不存在。案发现场已被破坏。

##### 第五步：解压快照（生成新目录）

注意心智模型：我们要把备份恢复到一个**全新的空目录**（例如 `/var/lib/etcd-restore`），而不是直接覆盖现有的 `/var/lib/etcd`。

```bash
etcdutl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore
```

执行完毕后，你可以 `ls -l /var/lib/etcd-restore`，会发现里面生成了新的数据文件。

##### 第六步：大脑移植（切换 etcd 数据目录）

在 kubeadm 集群中，etcd 是以 Static Pod（静态 Pod）的形式运行的。kubelet 会死死盯着 `/etc/kubernetes/manifests/` 目录。只要我们修改了里面的 yaml，kubelet 就会自动重启对应的 Pod。

```bash
vi /etc/kubernetes/manifests/etcd.yaml
```

翻到文件最下面，找到 `volumes` 挂载部分的 `hostPath`。 **将原来的 `/var/lib/etcd` 修改为 `/var/lib/etcd-restore`**：

```yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-restore  # <--- 修改这里！
      type: DirectoryOrCreate
    name: etcd-data
```

保存并退出 (`:wq`)。

##### 第七步：见证奇迹

保存退出后，kubelet 会发现 yaml 发生了变化，开始销毁旧的 etcd pod 并挂载新目录启动新的 etcd。这个过程大概需要 30 秒到 1 分钟。在这期间，你敲 `kubectl` 可能会卡住或报错（因为 apiserver 连不上数据库了），这是正常现象。

耐心等待一小会儿，再次输入：

```bash
kubectl get pods -n disaster-recovery
```

你会发现，刚才被删除的 `disaster-recovery` 命名空间和里面的 `survivor-pod` 又奇迹般地复活了！这就标志着你的集群时光倒流成功。

##### 总结

常用命令：

**数据备份**：`etcdctl snapshot save backup.db`

**查看备份文件哈希和体积**： `etcdutl snapshot status /opt/etcd-backup.db -w table`

**备份恢复到指定路径：**

```bash
etcdutl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore
```