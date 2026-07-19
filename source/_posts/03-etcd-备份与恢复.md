---
title: 03-ETCD 备份与恢复
tags: []
id: '215'
categories:
  - - k8s 实验
date: 2026-03-05 07:11:14
---

#### 实验准备（制造测试数据）

在备份前，先在集群中创建一点数据，用于后续验证恢复效果：

```bash
kubectl create namespace etcd-test
kubectl run test-pod --image=nginx -n etcd-test
```

* * *

#### 阶段一：执行快照备份 (Backup)

**1\. 确认参数并执行备份** 备份必须携带三个证书，并且务必加上 `ETCDCTL_API=3` 环境变量。考场上快照的目标路径会由题目指定（例如 `/opt/etcd-backup.db`）。

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

**2\. 验证快照（考场好习惯）** 确认文件无损且正确生成：

```bash
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/etcd-backup.db
```

_(此时可以执行 `kubectl delete namespace etcd-test` 删掉刚才的数据，模拟灾难发生。)_

* * *

#### 阶段二：执行快照恢复 (Restore)

**1\. 收集集群启动参数** 为了防止恢复后 ETCD 报错 `Member mismatch`，必须先从原配置中查出集群参数：

```bash
cat /etc/kubernetes/manifests/etcd.yaml  grep "initial-cluster" -A 2 -B 2
```

**2\. 确保目标目录绝对干净（防存在报错）** 恢复的目标目录（如 `/var/lib/etcd-restore`）**绝对不能提前存在**。：

```bash
sudo rm -rf /var/lib/etcd-restore
```

**3\. 执行完全体恢复命令** 结合第 1 步查到的参数，将快照数据解压到全新的目录中：

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --name=master \
  --data-dir=/var/lib/etcd-restore \
  --initial-cluster=master=https://192.168.44.121:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://192.168.44.121:2380
```

_(看到 `added member...` 即代表数据落盘成功。)_

* * *

#### 阶段三：修改静态 Pod 配置并生效

**1\. 修改 ETCD 配置文件（最易错点）** 打开静态 Pod 的 YAML 文件：

```bash
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

*   **绝对不要碰** `volumeMounts` 里的 `mountPath`（那是容器内的路径，必须保持 `/var/lib/etcd`）。
*   直接跳到文件最底部的 `volumes:` 块。
*   **只修改 `hostPath`**（这是物理机上的真实路径），将其指向你刚才恢复的新目录：

```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-restore   # <--- 修改这里！
      type: DirectoryOrCreate
    name: etcd-data
```

**2\. 保存退出，等待重连** 保存退出（`:wq`）后，`kubelet` 会自动发现配置变化并重启 ETCD。 如果想加速这个过程，可以使用“移出-移入”神技：

```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/ && sleep 10 && sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

**3\. 最终验证** 等待约 30-60 秒，API Server 重新上线后，查询你的测试数据：

```bash
kubectl get pods -n etcd-test
```

#### 清理

1、清理 kubernetes 测试资源

```bash
kubectl delete namespace etcd-test
```

2、还原 ETCD 配置文件（最关键！）

为了下次做实验时不产生混乱，我们需要把 ETCD 的数据目录重新指回默认的 `/var/lib/etcd`。

打开 ETCD 的静态 Pod 配置文件：

```bash
sudo vi /etc/kubernetes/manifests/etcd.yaml
```

翻到文件最底部的 `volumes:` 区域，把刚才修改的 `hostPath.path` 改回原样：

```yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd          # <--- 把带有 -restore 的后缀删掉，恢复成默认路径
      type: DirectoryOrCreate
    name: etcd-data
```

保存并退出 (`:wq`)。

3、强制重启 ETCD 并等待 API Server 恢复

老规矩，用移出移入大法让 Kubelet 立刻加载默认路径的原始数据：

```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/ && sleep 10 && sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

等待大约 30 秒到 1 分钟，直到你能正常执行 `kubectl get nodes` 且不报错为止。这说明 API Server 已经重新连上了原始的 ETCD 数据库。

3、清理物理机上的残留文件和目录

现在原有的 ETCD 已经正常接管了，我们可以放心地把刚才实验产生的文件全部删掉：

```bash
## 删除备份的快照文件
sudo rm -f /opt/etcd-backup.db

## 删除我们练习恢复时创建的那个目录
sudo rm -rf /var/lib/etcd-restore
```