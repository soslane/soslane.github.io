---
layout:       post
title:        "RAG如何提高召回率"
author:       "Gweek"
catalog:      true
tags:
    - 人工智能
---

# 一、用「语义结构化文档」替代原始文档（最重要）

很多人直接上传：

* PDF
* Word
* 网页爬虫内容

但这些文本通常是：

* 标题不清
* 语义块混乱
* 表格拆碎

RAG检索时会出现：

```
问题 → 找不到对应语义块
```

### 正确做法

先把文档结构化：

```
# Docker Compose是什么

Docker Compose 是一个用于定义和运行多容器 Docker 应用的工具。

## 使用步骤
1 安装 Docker
2 创建 compose.yml
3 运行 docker compose up

## 常见问题
Q: docker compose 和 docker run 有什么区别
A: docker compose 用于多容器编排
```

这种结构有三个好处：

1️⃣ 标题可检索
2️⃣ 段落语义完整
3️⃣ FAQ命中率高

实际项目里：

**仅这一项就能让召回率提高 50%+**

---

# 二、Chunk切分策略优化（90%的人切错）

很多教程说：

```
chunk size = 500
```

这是非常粗糙的。

### 推荐策略

技术文档：

```
chunk size = 300~400
overlap = 50
```

FAQ：

```
chunk size = 150~200
overlap = 30
```

原因：

* 太大 → 检索不精准
* 太小 → 上下文丢失

最佳结构：

```
标题
+ 一段解释
+ 一个例子
```

这样一段就是一个 chunk。

---

# 三、混合检索（Hybrid Search）

只用向量检索会出现问题：

```
用户问题：docker启动
知识库：docker run
```

向量模型可能没匹配到。

解决方案：

```
向量检索 + 关键词检索
```

在 Dify 里：

```
检索模式：混合检索
```

流程变成：

```
问题
↓
向量搜索
关键词搜索
↓
合并结果
```

召回率通常提升：

```
30% ~ 80%
```

---

# 四、Query Rewrite（查询重写）

用户的问题往往很短：

```
docker compose?
```

但知识库是完整句：

```
Docker Compose 是什么
```

所以需要 AI 先改写问题。

例：

用户输入：

```
docker compose?
```

改写成：

```
What is docker compose and how to use it
```

或者：

```
docker compose 是什么
```

这样 embedding 检索更容易命中。

很多系统称为：

```
Query Expansion
```

---

# 五、增加召回层（Multi-Retrieval）

普通 RAG 是：

```
问题 → 检索一次
```

高级 RAG 是：

```
问题
↓
改写问题
↓
生成3个子问题
↓
分别检索
↓
合并结果
```

例如：

原问题：

```
docker compose怎么部署项目
```

AI生成：

```
docker compose 是什么
docker compose 部署步骤
docker compose 示例
```

分别检索。

最后合并。

这种方法在企业RAG里很常见。

召回率提升：

```
2~3倍
```

---

# 六、加入Rerank模型（非常关键）

很多系统只做：

```
向量搜索 → TopK
```

但向量搜索排序不一定准。

解决方案：

加入 **Rerank模型**。

流程：

```
向量检索 → Top10
↓
Rerank重新排序
↓
取Top3
```

Rerank模型会判断：

```
问题 与 文档 的真实相关性
```

常用模型：

* bge-reranker-large
* cohere-rerank

效果：

```
准确率 +40%
```

---

# 七、TopK动态策略（进阶技巧）

很多人固定：

```
TopK = 3
```

但其实应该：

```
先召回10
再重排
最后选3
```

流程：

```
向量检索 → Top10
关键词检索 → Top10
↓
合并
↓
Rerank
↓
取3
```

这样：

* 召回更广
* 结果更准

---

# 八、Embedding模型选择（经常被忽视）

如果是中文知识库：

建议：

```
bge-large-zh
```

而不是：

```
text-embedding-3-small
```

因为：

中文语义能力差别很大。

在中文文档中：

```
bge 召回率通常高20%+
```

---

# 九、真实项目的最佳配置（推荐）

如果你在 Dify 里做知识库：

推荐配置：

```
Embedding模型：bge-large-zh
索引模式：高质量
检索模式：混合检索

chunk size：350
overlap：50

召回数量：8
重排模型：bge-reranker-large

最终引用：3
相似度阈值：0.75
```

---
