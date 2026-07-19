---
title: Word2Vec
tags: []
id: '344'
categories:
  - - uncategorized
date: 2026-03-24 11:46:01
---

`Word2Vec` 其实就是一个**没有隐藏层激活函数**的 2 层神经网络（`浅层神经网络`），用来将单词映射为**低维稠密向量**。

* * *

### 1\. 核心原理

基于**分布假设**。通过大量文本语料学习词语的共现规律，将语义相似的词映射到向量空间中彼此靠近的位置

> 分布假设
> 
> "A word is characterized by the company it keeps."
> 
> （一个词是由它周围的词决定的。）

* * *

`Word2Vec` 包含两种浅层神经网络结构，核心目标是学习词向量，输入层到隐藏层为`词嵌入映射`，隐藏层到输出层为`概率预测`。

![](/wp-uploads/2026/03/1774352468-image.png)

##### 2.1 CBOW（Continuous Bag-of-Words，连续词袋模型）

![](/wp-uploads/2026/03/1774352492-image-1024x672.png)

*   **核心任务**：用**上下文词**预测**中心词**
*   **特点**：计算效率高，适合**大规模语料**；对高频词效果更优
*   **类比**：像 “_完形填空_”，根据周围词语推测中间缺失的词

##### 2.2 Skip-gram（跳字模型）

![](/wp-uploads/2026/03/1774352527-image-1024x805.png)

*   **核心任务**：用**中心词**预测**上下文词**
*   **特点**：对**低频词、生僻词**效果更好；适合小数据集
*   **类比**：像 “联想扩散”，由一个词联想到周围相关的词

> 💡 **经验**：默认首选 `Skip-Gram`，除非数据量极大且追求速度。

* * *

### 3\. 训练优化技巧

为解决**大规模语料**下的计算瓶颈，`Word2Vec` 引入两大核心优化，大幅提升训练效率。

##### 0\. 🔑 关键点：我们要的是什么？

训练完成后：

*   **输出层权重**：扔掉 ❌
*   **投影层权重矩阵**：保留 ✅

这个 **V×N 的矩阵**，**每一行就是一个词的向量**

##### 1\. 负采样（Negative Sampling）⭐ 最常用

*   **思想**：不需要更新所有词的权重，只更新**正确的词**和**几个错误的词**。
*   **原理**：将多分类问题转化为**二分类任务**
*   **操作**：对每个正样本（中心词 + 真实上下文词），随机采样 K 个负样本（中心词 + 噪声词）
*   **优势**：仅更新目标词与采样词的参数，避免全词表计算，**训练速度提升显著**

##### 2\. 层次 Softmax（Hierarchical Softmax）

*   **原理**：用 `Huffman树` 组织词汇表，将根节点到叶节点的 `路径概率乘积` 作为预测结果，类似于`二叉搜索`
*   **优势**：将时间复杂度从 `O (V)` 降至 `O (logV)`，V 为词汇量，适合**超大词汇表**

* * *

### 4\. 局限性与发展

##### 4.1 主要局限

*   **静态向量**：一词一向量，无法处理**一词多义**（如 “bank” 既指银行也指河岸）
*   **忽略词序**：依赖固定窗口，无法捕捉**长距离语义依赖**和**词序信息**
*   **未登录词 OOV**：无法处理训练语料外的新词（如新兴术语、专有名词）
*   **上下文无关**：同一词在不同语境下向量相同，语义表达不够精细

##### 4.2 后续演进

*   **FastText**：引入子词信息，支持未登录词生成
*   **GloVe**：结合全局统计与局部上下文，提升语义表达
*   **Transformer 系列**：动态上下文编码，解决一词多义与长距离依赖问题（如 BERT、GPT）

* * *

### Word2Vec 示例代码

安装依赖库：`pip install gensim nltk`

import gensim
from gensim.models import Word2Vec
from gensim.utils import simple\_preprocess
import nltk
from nltk.corpus import stopwords

## ---------------------- 1. 数据准备与预处理 ----------------------
## 下载nltk停用词（仅首次运行需要）
nltk.download('stopwords')
stop\_words = stopwords.words('english')

## 示例语料（可替换为你自己的文本文件/大规模语料）
corpus = \[
    "Natural language processing is a subfield of artificial intelligence",
    "Word2Vec is a popular technique for word embedding in NLP",
    "Word embedding converts words into numerical vectors",
    "Semantic similarity can be measured using word vectors",
    "King minus man plus woman equals queen in word2vec",
    "Tokyo is the capital of Japan, Beijing is the capital of China",
    "Machine learning models use word vectors as input features",
    "Cat likes eating fish and dog loves enjoying bones"
\]

## 文本预处理函数：分词 + 去除停用词 + 过滤短词
def preprocess(text):
    # 分词（gensim内置的简单分词，适合英文）
    tokens = simple\_preprocess(text, deacc=True)  # deacc=True去除标点
    # 过滤停用词和长度<2的词
    tokens = \[token for token in tokens if token not in stop\_words and len(token) > 2\]
    return tokens

## 对语料库进行预处理，得到训练用的句子列表（每个句子是分词后的token列表）
processed\_corpus = \[preprocess(sentence) for sentence in corpus\]
print("预处理后的语料：")
for sent in processed\_corpus:
    print(sent)

## ---------------------- 2. 训练Word2Vec模型 ----------------------
## 核心参数说明（新手重点关注）：
## - vector\_size：词向量维度（常用50/100/300，小语料用50即可）
## - window：上下文窗口大小（中心词左右各window个词）
## - min\_count：最小词频，低于该值的词忽略（过滤低频噪声）
## - sg：训练模型类型，sg=0为CBOW，sg=1为Skip-gram
## - workers：并行训练线程数（根据CPU核心数调整）
model = Word2Vec(
    sentences=processed\_corpus,
    vector\_size=50,        # 词向量维度
    window=3,              # 上下文窗口
    min\_count=1,           # 保留所有出现过的词（小语料专用）
    sg=1,                  # 使用Skip-gram模型（对小语料更友好）
    workers=4,             # 4线程训练
    epochs=100             # 训练轮数（小语料需增加轮数保证效果）
)

## 保存模型（可选，后续可直接加载）
model.save("word2vec\_demo.model")
## 加载模型（后续使用时）：model = Word2Vec.load("word2vec\_demo.model")

## ---------------------- 3. 模型使用示例 ----------------------
## 3.1 查看单个词的向量
print("\\n=== 查看'plus'的词向量（前10维）===")
print(model.wv\['plus'\]\[:10\])  # wv是word vector的缩写，存储所有词向量量

## 3.2 计算两个词的语义相似度
print("\\n=== 计算语义相似度 ===")
print(f"like vs love: {model.wv.similarity('like', 'love'):.4f}")
print(f"beijing vs tokyo: {model.wv.similarity('beijing', 'tokyo'):.4f}")
print(f"king vs queen: {model.wv.similarity('king', 'queen'):.4f}")
print(f"china vs japan: {model.wv.similarity('china', 'japan'):.4f}")

## 3.3 找最相似的词
print("\\n=== 找与'like'最相似的5个词 ===")
similar\_words = model.wv.most\_similar('like', topn=5)
for word, score in similar\_words:
    print(f"{word}: {score:.4f}")

## 3.4 语义类比（国王-男人+女人=王后）
print("\\n=== 语义类比：king - man + woman == ? ===")
try:
    analogy = model.wv.most\_similar(positive=\['king', 'woman'\], negative=\['man'\], topn=1)
    print(f"结果：{analogy\[0\]\[0\]} (相似度：{analogy\[0\]\[1\]:.4f})")
except KeyError as e:
    print(f"类比失败：缺少词 {e}（语料太小导致）")

**参数调整**：

*   大规模语料：可将`vector_size`调至 100/300，`min_count`调至 5/10（过滤低频词），`epochs`调至 10-20；
*   小语料：保持 `min_count=1`，增加 `epochs`（如 100）保证训练效果。

**如果结果奇怪**：  
_根本原因：语料库太小了_  
直接用别人训练好的模型，不要自己训练小语料：

from gensim.models import KeyedVectors

## 加载 Google 预训练的 Word2Vec (300 维，300 万词)
## 下载地址：https://code.google.com/archive/p/word2vec/
model = KeyedVectors.load\_word2vec\_format('GoogleNews-vectors-negative300.bin', binary=True)

## 现在测试结果会正常！
print(model.wv.similarity('like', 'love'))  # 应该 > 0.5
print(model.wv.similarity('king', 'queen'))  # 应该 > 0.6

## 类比也会正确
result = model.wv.most\_similar(positive=\['king', 'woman'\], negative=\['man'\])
print(result\[0\]\[0\])  # 应该输出 'queen'

* * *