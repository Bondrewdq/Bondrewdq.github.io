---
title: 循环神经网络 RNN
tags: []
id: '351'
categories:
  - - uncategorized
date: 2026-03-24 11:54:06
---

### 背景及推导

**需求**：输入一句话，输出每个单词的褒贬性

![](/wp-uploads/2026/03/1774352814-image-1024x509.png)

假设对于上面 5 个单词，每个单词采用 300 维的 `词向量`，那么输入端就需要 1500 个词向量，这会带来几个**缺点**：

1.  输入层的**节点太多**了，并且是**变长**的，会随着句子的长短发生变化
2.  无法体现词语之间的**先后顺序**，仅仅只是把他们拉直展开成一个大向量，直接送入输入层（类似于 MLP）

为了解决上述缺点，**可以让上一步的信息参与下一步的运算**，具体可以这样做：

![](/wp-uploads/2026/03/1774352850-image-1024x521.png)

![](/wp-uploads/2026/03/1774352856-image.png)

*   第一个词 $X^{<1>}$ 经过非线性变换 g($\\cdot$) 后得到一个中间结果(`隐藏状态`) $h^{<1>}$ ，它再经过一次非线性变换得到第一个输出 $Y^{<1>}$
*   接着，第二个词 $X^{<2>}$ 与刚才的隐藏状态 $h^{<1>}$ 一起参与非线性变换，得到隐藏状态 $h^{<2>}$ ，它再经过一次非线性变换得到 $Y^{<2>}$

*   g($\\cdot$)：_激活函数，一般取双曲正切 tanh_
*   _$W\_{xh}$：专门针对词向量的矩阵_
*   $W\_{hh}$：_专门针对隐藏状态的矩阵_
*   $W\_{hy}$：_专门针对输出结果的矩阵_
*   $b$：_偏置项 bias_

上图流程可以简化为下面两种等效表达方式：

![](/wp-uploads/2026/03/1774352905-image-1024x507.png)

![](/wp-uploads/2026/03/1774352921-image-1024x377.png)

> _参考链接：[语言居然可以被计算出来？从 RNN 到 Transformer【AI入门05】\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1MNoRYEEVM/?spm_id_from=333.1387.favlist.content.click&vd_source=c5551749b3e52456db43941bf55d8d6b)_

* * *

### RNN 的问题：梯度消失/爆炸

![](/wp-uploads/2026/03/1774352959-image.png)

**反向传播时**，梯度需要从后往前传：

∂L∂W\=∂L∂sn×∂sn∂sn−1×∂sn−1∂sn−2×...×∂s1∂W\\frac{\\partial L}{\\partial W}=\\frac{\\partial L}{\\partial s\_n}\\times\\frac{\\partial s\_n}{\\partial s\_{n-1}}\\times\\frac{\\partial s\_{n-1}}{\\partial s\_{n-2}}\\times ... \\times\\frac{\\partial s\_1}{\\partial W}

*   如果 $\\frac{\\partial s\_t}{\\partial s\_{t-1}} < 1$（如 0.5），连乘 100 次 → `0.5^100 ≈ 0` → **梯度消失** → 前面的权重几乎不更新 → 模型**记不住长距离信息**
*   如果 $\\frac{\\partial s\_t}{\\partial s\_{t-1}} > 1$（如 1.5），连乘 100 次 → `1.5^100 ≈ 亿` → **梯度爆炸** → 权重变成 `NaN` → 模型训练崩溃

**解决方案**：  
原始的 RNN 几乎已被淘汰

方法

说明

**梯度裁剪**

限制梯度最大值（如 1.0）

**ReLU 激活**

缓解梯度消失（但 RNN 常用 tanh）

**LSTM/GRU**

**最有效的方案** ⭐ LSTM 和 GRU 中增加了一种名为“门”的结构

> Gated RNN
> 
> 在 RNN 的学习中，`梯度消失` 也是一个大问题。为了解决这个问题，需要从根本上改变 RNN 层的结构。人们已经提出了诸多 Gated RNN 框架，其中具有代表性的有 [LSTM](https://iplusjia.top/2026/03/24/%e9%95%bf%e7%9f%ad%e6%9c%9f%e8%ae%b0%e5%bf%86-lstm/) 和 `GRU`

* * *

### PyTorch 代码示例

基础 RNN：

import torch
import torch.nn as nn

class SimpleRNN(nn.Module):
    def \_\_init\_\_(self, input\_size, hidden\_size, num\_classes):
        super().\_\_init\_\_()
        self.rnn = nn.RNN(input\_size, hidden\_size, batch\_first=True)
        self.fc = nn.Linear(hidden\_size, num\_classes)
    
    def forward(self, x):
        # x: (batch, seq\_len, input\_size)
        out, hidden = self.rnn(x)  # out: (batch, seq\_len, hidden)
        # 取最后一个时间步
        out = self.fc(hidden\[-1\])
        return out

## 使用
model = SimpleRNN(input\_size=100, hidden\_size=128, num\_classes=2)

用上面的 model，结合 `IMDB 数据集`进行情感分析：

\# 超参数
batch\_size = 64
seq\_len = 100
learning\_rate = 0.001

## 损失和优化器
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=learning\_rate)

## 训练
for epoch in range(10):
    model.train()
    for batch\_x, batch\_y in train\_loader:
        optimizer.zero\_grad()
        output = model(batch\_x)
        loss = criterion(output, batch\_y)
        loss.backward()
        
        # 梯度裁剪（防止 RNN 梯度爆炸）⭐
        torch.nn.utils.clip\_grad\_norm\_(model.parameters(), 1.0)
        
        optimizer.step()
    
    # 验证
    model.eval()
    val\_acc = evaluate(model, val\_loader)
    print(f"Epoch {epoch}: Val Acc = {val\_acc:.4f}")

* * *