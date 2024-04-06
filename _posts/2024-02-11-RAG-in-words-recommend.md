---
title: RAG在用户输入推荐场景中的应用
date: 2024-02-11 22:01:39
categories:
- 人工智能
tags:
- 后端
---

## 推荐系统的实现方式

推荐系统有多种实现方式, 简单的可以直接使用类似于ElasticSearch这样的搜索引擎, 通过检索的方式获取到相关的信息, 然后返回给用户, 也可以使用深度学习模型, 通过训练的方式获取到相关的信息, 然后返回给用户.

## ElasticSearch实现推荐系统

ElasticSearch是一个基于Lucene的搜索引擎, 提供了一个分布式多用户能力的全文搜索引擎, 通过ElasticSearch可以实现搜索, 分析, 存储等功能, 适用于各种场景, 包括推荐系统.

ElasticSearch提供了MLT(More Like This)的功能, 可以通过检索的方式获取到相关的信息, 并进行相应的评分排序返回给调用者, 用以实现简单的推荐系统.  
ElasticSearch内部的评分排序算法是基于BM25的, 通过计算文档之间的相似度, 然后进行排序. 但是结果中存在噪音数据的可能性, 所以在使用时需要根据实际情况通过参数min_doc_freq和min_term_freq进行调整.

### 样例

```
POST news/_search
{
  "query": {
    "more_like_this": {
      "fields": [
        "title"
      ],
      "like": [
        "体育运动"
      ],
      "analyzer": "ik_smart",
      "min_doc_freq": 2,
      "min_term_freq": 1
    }
  }
}
```

## 什么是RAG

RAG是检索增强生成(Retrieval Augmented Generation)的缩写, 是一种结合检索和生成的模型, 通过检索的方式获取到相关信息, 然后通过生成的方式生成出最终的结果, 适用于问答系统, 文本生成等场景.

## 为何要用RAG

在传统的用户输入推荐场景中, 常常使用的是检索的方式, 通过检索用户输入的关键词, 然后返回相关的结果, 但是这种方式往往会受限于检索的质量, 无法提供足够的信息量, 而RAG模式则可以通过生成的方式, 生成出更加丰富的信息, 从而提升用户体验.

## RAG的实现方式

### 检索

在RAG实现的第一步, 需要通过检索的方式获取到相关的信息, 目前常用的检索方式有文本检索和语义检索.

#### 文本检索

1. 文本预处理: 对文本进行分词, 去除停用词等处理;
2. 文本索引: 将处理后的文本建立倒排索引, 记录包含每个词的文档以及位置重新洗, 以便后续检索;
3. 文本检索: 通过用户输入的关键词, 在倒排索引中检索相关的文档, 并根据TFIDF或者BM25等算法进行排序返回结果.

可以通过使用ElasticSearch等搜索引擎内置的相似检索功能获得相应数据.

#### 语义检索

语义检索是通过将文本转换为向量表示, 然后通过计算向量之间的相似度, 从而获取到相关的信息, 例如BERT, GPT等模型. 使得用户可以获得更加丰富的信息.
此处以m3e模型为例, 通过检索的方式获取到相关的信息, 并进行相应的评分排序返回给调用者, 用以实现推荐系统.

##### 文本切分

如果文本块过大, 可能会影响检索的效果, 可以通过文本切分的方式将文本切分为多个子文本, 然后分别进行检索, 最后将结果进行合并返回给用户.  
通常文本切分的方式有按句子切分, 按段落切分, 按TOKEN切分等.

##### 文本嵌入

1. 词嵌入: 将文本中的词转换为向量表示, 例如Word2Vec, GloVe等;
2. 句子嵌入: 将文本中的句子转换为向量表示, 例如BERT, GPT等;
3. 文档嵌入: 将文本中的文档转换为向量表示, 例如Doc2Vec等.

##### 语义检索

1. 计算相似度: 通过计算向量之间的相似度, 从而获取到相关的信息, 例如余弦相似度等;
2. 排序: 根据相似度进行排序, 返回给用户.

##### 示例代码

```python
from sentence_transformers import SentenceTransformer, util

model = SentenceTransformer('moka-ai/m3e-small')
recommends = []

# 输入文本
query_list = ["体育运动", "娱乐活动"]

# 检索相关文档
corpus = ["体育运动", "足球比赛", "篮球比赛", "乒乓球比赛", "音乐会", "电影节"]
corpus_embeddings = model.encode(corpus, normalize_embeddings=True)
query_embedding = model.encode(query, normalize_embeddings=True)

# 计算相似度
for i, corpus_embedding in enumerate(corpus_embeddings):
    score = corpus_embedding @ query_embedding.T
    max_score = score.argsort()[-1]
    recommends.append(corpus[max_score])
```

> M3E是一种基于BERT的语义检索模型, 通过将文本转换为向量表示, 然后通过计算向量之间的相似度, 从而获取到相关的信息, 适用于问答系统, 文本生成等场景.

#### 文本检索和语义检索的对比

文本检索的通过匹配关键词的方式, 无法处理一些复杂的语义关系, 例如同义词, 近义词等, 而语义检索则可以通过语义模型获取到更加丰富的信息, 但是语义检索的计算成本较高, 且需要大量的数据进行训练, 文本检索基于索引完成, 速度更快.

#### 文本检索和语义检索的结合

为了兼具两者的优点, 可以将文本检索和语义检索结合起来, 通过文本检索获取到相关的文档, 然后通过语义检索获取到更加丰富的信息, 从而提升检索的效果.

#### 多路召回

如果只采用单一的检索方式, 可能会受限于检索的质量, 无法提供足够的信息量, 所以可以通过多路召回的方式, 将多种检索方式的结果进行合并, 从而提升检索的效果.

多路召回通常采用的方式有并集召回, 交集召回, 加权召回等.

例如通过给不同的检索方式设置不同的权重, 从而获取到更加丰富的信息, 例如文本检索的权重为0.6, 语义检索的权重为0.4, 从而提升检索的效果.

而按照段落/句子/页不同的角度进行语义编码检索, 综合也可以获得更加丰富的信息.

#### 重排序

在多路召回的方式中, 可能会存在一些噪音数据, 为了提升检索的效果, 可以通过重排序的方式, 对检索出来的top k结果进行重新排序, 从而提升检索的效果.
例如综合使用BM25+BGE RERANK的方式, 通过BM25获取到相关的文档, 然后通过BGE RERANK对文档进行重排序, 从而提升检索的效果.

重排序的模型有很多, 例如基于BERT的bge reranker等

样例代码

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

tokenizer = AutoTokenizer.from_pretrained("BAAI/bge reranker-base")
model = AutoModelForSequenceClassification.from_pretrained("BAAI/bge reranker-base")

# 多路召回的结果
retrieval_results = ["体育运动", "足球比赛", "篮球比赛", "乒乓球比赛", "音乐会", "电影节"]

# 重排序
rerank_results = []
for result in retrieval_results:
    inputs = tokenizer(result, return_tensors="pt")
    outputs = model(**inputs)
    rerank_results.append(outputs)

# 对结果进行重排序
rerank_results = sorted(rerank_results, key=lambda x: x["score"], reverse=True)
```

### 大模型生成推荐结果

在检索出相应的文档后, 可以通过大模型生成推荐结果, 例如GPT, CHATGLM等模型, 通过配合一定的Prompt, 将检索结果和用户输入结合生成出更加丰富的信息, 从而提升用户体验.  
同时, 可以在prompt中加入一些特定的关键词, 从而引导模型生成出更加符合用户需求的信息, 并且根据不同的情况返回一些特点的关键词, 例如无法找到结果, 可以返回一些提示信息, 便于后续的处理.

#### 过滤敏感信息

在生成推荐结果时, 可能会存在一些敏感信息, 为了保护用户隐私, 可以通过过滤敏感信息的方式, 对生成的结果进行过滤, 从而保护用户隐私.

#### 过滤无效信息

在输出的结果后处理流程中, 如果发现提前设定的关键词出现, 可以返回指定的提示信息, 例如"没有找到相关信息", 从而提升用户体验.

## 总结

RAG的实际应用场景可以分成如下步骤: 
1. 文本检索
2. 语义检索
3. 多路召回
4. 重排序
5. 大模型生成推荐结果
6. 过滤敏感信息
7. 过滤无效信息
