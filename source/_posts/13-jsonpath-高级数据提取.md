---
title: 13-JSONPath 高级数据提取
tags: []
id: '253'
categories:
  - - k8s 实验
date: 2026-03-11 09:10:26
---

#### 实验前置准备：制造多样化数据

先拉起几个不同状态和镜像的 Pod，为我们的查询提供素材。

```bash
kubectl run pod-nginx --image=nginx
kubectl run pod-httpd --image=httpd
kubectl run pod-fail --image=busybox -- false  # 故意让它 Error/CrashLoopBackOff
```

等几秒钟，确保它们状态稳定。

* * *

#### 核心技术逻辑：如何找路径？

这是掌握 JSONPath 的唯一正确方法：**先输出纯 JSON，看清结构再写路径。**

执行：`kubectl get pods -o json more` 你会看到最外层是一个巨大的 JSON 对象，包含一个 `items` 数组。每个 Pod 都是 `items` 里的一个元素（Go 里的 Slice）。

*   `$`：代表整个 JSON 根节点（在 kubectl 中可以省略）。
*   `.`：代表获取子节点（Struct 的字段）。
*   `[]`：代表操作数组。`[*]` 表示遍历整个切片。

* * *

#### 实验过程

##### 阶级一：基础提取（遍历数组）

**目标：** 获取当前命名空间下所有 Pod 的名称。

既然我们知道数据都在 `items` 数组里，而名字在 `metadata.name` 里，命令就是：

```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

> **预期输出：** `pod-fail pod-httpd pod-nginx` （所有名字连在了一起，中间有空格）。

* * *

##### 阶级二：高级格式化（CKA 必考的 `range` 循环）

刚才的输出都在同一行，很不优雅，而且无法同时输出多个字段。K8s 的 JSONPath 扩展了 Go Template 的 `range` 语法，这是考场上的一击必杀技。

**目标：** 按行输出每个 Pod 的名称和它的 IP 地址，中间用 Tab 键隔开。

**命令解析：**

*   `{range .items[*]}`：开启一个 Go 切片循环。在循环内部，当前上下文就变成了单个 Pod 对象。
*   `{.metadata.name}`：获取名字。
*   `{"\t"}`：插入一个 Tab 制表符。
*   `{.status.podIP}`：获取 IP。
*   `{"\n"}`：插入一个换行符。
*   `{end}`：结束循环。

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

> **预期输出：** pod-fail (可能为空，因为它挂了) pod-httpd 192.168.x.x pod-nginx 192.168.y.y

* * *

##### 阶级三：条件过滤（Filter 表达式）

这是最高阶的用法。有时候题目要求只提取满足特定条件的资源。

**语法规则：** `?(@.字段 == "值")`

*   `?()` 是过滤器的固定语法。
*   `@` 代表当前正在遍历的元素自身。

**目标：** 只提取状态处于 `Running` 的 Pod 名称。

我们要在遍历 `items` 数组时加上条件拦截：

```bash
kubectl get pods -o jsonpath='{.items[?(@.status.containerStatuses[0].ready==true)].metadata.name}'
```

> **预期输出：** 只有 `pod-httpd pod-nginx`，那个处于 Error/CrashLoopBackOff 的 `pod-fail` 被完美过滤掉了。

* * *

##### 阶级四：资源排序（Sort-by）

排序其实不属于 `-o jsonpath` 的范畴，但经常与数据提取结合使用。它的参数直接接受一个 JSONPath 表达式。

**目标：** 列出所有 Node 节点，并按它们的创建时间（`metadata.creationTimestamp`）进行排序。

```bash
kubectl get nodes --sort-by='.metadata.creationTimestamp'
```

_(由于你的 Master 和 Node1/Node2 是按顺序加入的，你会清楚地看到时间的先后排序)_

* * *

#### 🛠️ 清理战场

```bash
kubectl delete pod pod-nginx pod-httpd pod-fail
```