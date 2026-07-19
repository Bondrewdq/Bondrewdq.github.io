---
title: 长短期记忆 LSTM
tags: []
id: '360'
categories:
  - - uncategorized
date: 2026-03-24 12:05:53
---

长短期记忆（Long Short-Term Memory，`LSTM`） 是 [RNN](https://iplusjia.top/2026/03/24/%e5%be%aa%e7%8e%af%e7%a5%9e%e7%bb%8f%e7%bd%91%e7%bb%9c-rnn/) 最重要的变体，也是深度学习 NLP 发展史上的**里程碑**。在 Transformer 出现之前（2017 年），LSTM 是序列建模的绝对王者。

* * *

### 背景：RNN 梯度消失

> 句子："我住在法国，...（中间 200 个字）... 所以我会说\_\_\_"  
> 空格应该填 _“法语”_

**RNN 的问题**：  
需要记住 200 个字前的"法国"。`反向传播` 时，梯度要连乘 200 次，导致前面的信息传不过来。  
_如果每次乘 0.9 → `0.9^200 ≈ 0.000000001` → **梯度消失**_

* * *

### LSTM 的结构及接口

![](/wp-uploads/2026/03/1774353528-image-1024x403.png)

![](/wp-uploads/2026/03/1774353542-image.png)

LSTM 与 RNN 的接口的不同之处在于，LSTM 还有路径 `c`。这个 c 称为 `记忆单元`（或者简称为“单元”），相当于 LSTM 专用的记忆部门。

*   记忆单元的特点：**仅在 LSTM 层内部接收和传递数据**，对外部不可见，我们甚至不用考虑它的存在
*   $\\sigma$：`sigmoid函数`用于求门的开合程度（sigmoid函数的输出范围在0.0 ~ 1.0）

* * *

#### 输出门 Output Gate

*   **作用**：决定**输出**哪些信息作为隐藏状态 $h\_t$（求出隐藏状态 $h\_t$）
*   **公式**：$o\_t = \\sigma\\left(x\_t W\_x^{(o)} + h\_{t-1} W\_h^{(o)} + b^{(o)}\\right)$

`隐藏状态` $h\_t=o\_t ⊙ \\tanh(c\_t)$，记忆单元 $c\_t$ 和隐藏状态 $h\_t$ 的关系只是按元素应用 $\\tanh$ 函数。这意味着，记忆单元 $c\_t$ 和隐藏状态 $h\_t$ 的_元素个数相同_。

> ⊙ 表示 `阿达玛乘积`。即，对应元素的乘积

* * *

#### 遗忘门 Forget Gate

*   **作用**：决定从细胞状态中**丢弃**哪些信息
*   **公式**：$f\_t = \\sigma\\left(x\_t W\_x^{(f)} + h\_{t-1} W\_h^{(f)} + b^{(f)}\\right)$

* * *

#### 输入门 Input Gate

*   **作用**：决定**更新**哪些新信息到细胞状态
*   **公式**：$i\_t = \\sigma\\left(x\_t W\_x^{(i)} + h\_{t-1} W\_h^{(i)} + b^{(i)}\\right)$

* * *

#### 新的记忆单元

![](/wp-uploads/2026/03/1774353784-image-1024x712.png)

向记忆单元添加的`新信息` ：$\\hat{c\_t} = \\tanh\\left(x\_t W\_x^{(g)} + h\_{t-1} W\_h^{(g)} + b^{(g)}\\right)$  
`新的记忆单元`： $c\_t=f ⊙ c\_{t-1} + i ⊙ \\hat{c\_t}$，_细胞状态 = 遗忘 × 旧细胞 + 输入 × 新候选_

_$\\hat{c\_t}$ 也即上面的 g_

**这个 tanh 节点的作用不是门，而是将新的信息添加到记忆单元中**。因此，它不用 `sigmoid 函数` 作为激活函数，而是使用 `tanh 函数`。

* * *

### LSTM 不会梯度消失

观察记忆单元的反向传播：

![](/wp-uploads/2026/03/1774353862-image-1024x298.png)

记忆单元的反向传播仅流过 `+` 和 `×` 节点。

*   `+ 节点` 将上游传来的**梯度原样流出**，所以梯度没有变化（退化）
*   `× 节点` 的计算并**不是矩阵乘积**，而是对应元素的乘积（阿达玛积）

* * *

### PyTorch 代码示例

基础 LSTM：

import torch
import torch.nn as nn

class SimpleLSTM(nn.Module):
    def \_\_init\_\_(self, input\_size, hidden\_size, num\_layers, num\_classes):
        super().\_\_init\_\_()
        self.hidden\_size = hidden\_size
        self.num\_layers = num\_layers
        
        # LSTM 层
        self.lstm = nn.LSTM(input\_size, hidden\_size, 
                           num\_layers=num\_layers,
                           batch\_first=True,
                           dropout=0.3)  # 层间 Dropout
        
        # 全连接层
        self.fc = nn.Linear(hidden\_size, num\_classes)
    
    def forward(self, x):
        # x: (batch\_size, seq\_len, input\_size)
        
        # 初始化隐藏状态
        h0 = torch.zeros(self.num\_layers, x.size(0), self.hidden\_size)
        c0 = torch.zeros(self.num\_layers, x.size(0), self.hidden\_size)
        
        # 前向传播
        out, (hn, cn) = self.lstm(x, (h0, c0))
        # out: (batch, seq\_len, hidden\_size)
        # hn: (num\_layers, batch, hidden\_size)
        
        # 取最后一个时间步的输出
        out = self.fc(out\[:, -1, :\])
        return out

## 使用
model = SimpleLSTM(input\_size=100, hidden\_size=128, 
                   num\_layers=2, num\_classes=2)

IMDB 情感分析:

import torch.optim as optim

## 超参数
batch\_size = 64
seq\_len = 100
learning\_rate = 0.001
epochs = 10

## 模型、损失、优化器
model = SentimentBiLSTM(vocab\_size=10000, embed\_dim=300,
                        hidden\_size=128, num\_classes=2)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=learning\_rate)

## 训练循环
for epoch in range(epochs):
    model.train()
    total\_loss = 0
    
    for batch\_x, batch\_y in train\_loader:
        # batch\_x: (64, 100), batch\_y: (64,)
        
        optimizer.zero\_grad()
        output = model(batch\_x)
        loss = criterion(output, batch\_y)
        loss.backward()
        
        # 梯度裁剪（防止 LSTM 梯度爆炸）⭐
        torch.nn.utils.clip\_grad\_norm\_(model.parameters(), 1.0)
        
        optimizer.step()
        total\_loss += loss.item()
    
    # 验证
    model.eval()
    correct = 0
    total = 0
    with torch.no\_grad():
        for batch\_x, batch\_y in val\_loader:
            output = model(batch\_x)
            predicted = torch.argmax(output, dim=1)
            correct += (predicted == batch\_y).sum().item()
            total += batch\_y.size(0)
    
    val\_acc = correct / total
    print(f"Epoch {epoch+1}: Loss={total\_loss:.4f}, Val Acc={val\_acc:.4f}")

* * *