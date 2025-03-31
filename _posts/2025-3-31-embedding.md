---
layout: post
title: Embedding 记录
tags: AI
excerpt: 随笔，整理中，doing...
---
&emsp;&emsp;`Embedding` 即向量化；

利用计算**相似性**，找最相似的相关对象：

&emsp;&emsp;`search` 搜索

&emsp;&emsp;`clustering` 聚类（最相似的一组）

&emsp;&emsp;`recommendations` 推荐（最相似的几个）

&emsp;&emsp;`anomaly detection` 异常检测（识别出相关性很小的异常值）

&emsp;&emsp;`diversity measurement` 多样性测量（分析相似性分布）

**限制**：时效性、安全性、token长度；

eg 用户 query 流程：
<p><img src="https://acceleratorssr.github.io/image/embedding.png" alt="用户 query 流程"></p>

1、**切片**：chunk

为什么需要切片：

&emsp;&emsp;1）模型对文本向量化的token长度是有上限的，token长度的限制主要是受限于计算复杂
切片：

&emsp;&emsp;2）token 越长意味着越高的相似性算法复杂度，资源消耗的越多，计算的耗时越长；

&emsp;&emsp;&emsp;&emsp;方案一：使用 chunksize 切割，明显容易截断上下文；

&emsp;&emsp;&emsp;&emsp;方案二：基于MD的标准格式进行切分，使用LLM将文本提取出关键词进行向量化（用户
query较短，如果和完整文本做相似性匹配时召回率估计很低），并且用LLM对query进行关键词提取；

&emsp;&emsp;注：也结合具体的业务场景分析，如菜谱的检索，可考虑向量化菜品的原料+菜品本身性质等方案；

2、**向量化**；

`Tokenization`：将文本切分为 token；如基于预定义的规则（注：分词是分 token 的子集），或者基于统计的方法+机器学习确定分词方法；

`word embedding`:

&emsp;&emsp;使用密集向量表示词语的方法，目标是捕获词语之间的语义联系；

&emsp;&emsp;如：矩阵分解方法、基于 `Graph` 的 embedding 方法、基于 `内容` 的 embedding 方法（如 word2vec[<sup>[1]</sup>](#1)（看上去有点过时了）、GloVe、FastText）和动态向量 embedding（如 ELMo、GPT、BERT）;

特点：

&emsp;&emsp;1、低维稠密向量：

&emsp;&emsp;&emsp;&emsp;每个词被表示为一个低维的稠密向量，几十到几百；

&emsp;&emsp;&emsp;&emsp;通过训练神经网络学习得到，可捕获语义上的相似性，相似的词在向量空间中更加接近；

&emsp;&emsp;2、上下文信息：

&emsp;&emsp;训练过程中会考虑上下文信息，可捕获词语的多种语义；



相似性匹配：

欧几里德距离（反应向量的绝对距离+适用于考虑向量长度的相似性计算）

余弦相似度（反应向量之间的夹角余弦值+适用于考虑向量方向的相似性计算，对长度不敏感，即适用于高维向量的相似性计算，语义搜索和分档分类等）

点积相似度（反应向量间的点积值，兼顾长度和方向，但是也因此对高维向量的相似性计算可能有误差）

**相似性搜索**：

- 并行化处理，将向量分桶，并行匹配；
- 降低向量维度；
- 缩小搜索范围，即利用聚类算法一次性过渡大部分距离较远的簇（如 K-Means[<sup>[2]</sup>](#2)）、HNSW图也同理；


3、相似性匹配；
4、向量数据库；





#### 备注
<ol>
    <li id="1">
    <strong>word2vec 训练模式</strong>：
    <ol>
        <li>
            CBOW：通过上下文来预测当前值，即一个句子中扣掉一个词，预测这个词是什么；
        </li>
        <li>
            Skip-grom：用当前值预测上下文，即反过来，词猜前后的词；
        </li>
    </ol>
    </li>
    <li id="2">
        <strong>K-Means</strong>：
    </li>



</ol>


> 碎碎念一下，wwww
> 回顾回顾 ssss：
> 1. 懒惰，没有榨干晚上下班后的时间用于刷算法，周末刷题的强度也低，高看自己 10min 左右做到 1a 的能力了；**治**：有空的时候多刷题保持题感，减少手残情况；
> 2. 胆小，面试时不能充分活动脑子，哪怕对于不确定的地方 也要给出自己的看法与解法，而不是语无伦次 不敢下推断；**治**：抽时间复习，加强自信，将熟悉的知识尽可能融会贯通；
> 3. 以周为粒度进行复盘，尽可能提早发现自己的不足，抓住任何可以提升能力的机会，多了解，多写代码，多思考，持之以恒 细水长流，在秋招之时 尽可能弥补和 高学历高技术力同学 间的差距 ：）
> 
> 抓住面试机会 wwww