---
layout:     post
title:     AI篇其三，Embedding
subtitle:   RAG的基础：Embedding、向量搜索和向量数据库。Vector search、Embedding
date:       2024-10-29
author:     catmai
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - langchain4j
    - AI
    -Milvus

---

# 开篇

拖更得有点久了，不知不觉langchain4j已经迎来了0.35.0版本。上回的demo如果遇到问题大概率是升级之后，ZhipuAiChatModel需要指定几个超时时间，代码需要变更为：

``` java
ZhipuAiChatModel chatModel = ZhipuAiChatModel.builder()
                .apiKey("")
                .logRequests(true)
                .logResponses(true)
                .maxRetries(1)
                .readTimeout(Duration.ofSeconds(30))
                .writeTimeout(Duration.ofSeconds(30))
                .connectTimeout(Duration.ofSeconds(30))
                .callTimeout(Duration.ofSeconds(30))
                .build();
```

那么，上一篇我们了解了智谱清言的API怎么通过langchain4j调用，这一篇一起来了解一个新的概念：**embedding**，中文翻译叫**嵌入**，这非常的抽象，非常难以理解。

# Embedding

什么是embedding？引用互联网上出现比较多的解释是：

> Embedding是一个相对低维的空间，可以将高维向量转换到其中。Embedding使得机器学习更容易在大规模的输入上进行，比如表示单词的稀疏向量。理想情况下，Embedding通过将语义相似的输入紧密地放置在Embedding空间中来捕获输入的一些语义。Embedding可以跨模型学习和重用。
>
> 原文链接：https://blog.csdn.net/thujiang000/article/details/122786518

按我的理解来说，他是一种更智能的检索，比如，把文本向量化后，运动、篮球、足球会有关联关系，而一般的拆分词检索是无法建立这样的关系的，比如elasticsearch中，检索运动关键字是给不出篮球、足球的信息的。但在向量数据库（embedding store）中，这是可行的，所以Embedding本身看起来就挺智能和聪明的了，这在很多推荐场景都使用到过，机器学习之类的也是必修的内容。

再拓展一点，有一些简单的人脸识别，也可以通过向量化来做，把图片信息通过embedding后，通过向量查询，可以查询到相似度较高的图片信息。当然想要人脸识别还没这么简单， 这里不展开说了。



# Embedding Store

向量存储，顾名思义，存储向量信息的地方，我这里选择milvus作为向量存储来使用，其实很多非关系型数据库也可以作为向量存储，例如elasticsearch，redis之类的。milvus是一个开源的向量数据库，既然是向量存储，还是选择向量数据库吧。同时，langchain4j也提供了mlivus相关的工具，使用起来比较方便。

示例代码：

```java
//初始化向量数据库
EmbeddingStore<TextSegment> embeddingStore = MilvusEmbeddingStore.builder()
                .host(MilvusConfig.getHost())//host信息
                .port(MilvusConfig.getPort())//端口信息
                .databaseName(MilvusConfig.getDatabaseName())//database信息
                .collectionName("test")//collection信息
                .dimension(384)
                .metricType(MetricType.L2)
                .build();
//确定向量模型
AllMiniLmL6V2EmbeddingModel embeddingModel = new AllMiniLmL6V2EmbeddingModel();
TextSegment segment1 = TextSegment.from("I like football.");
Embedding embedding1 = embeddingModel.embed(segment1).content();
embeddingStore.add(embedding1, segment1);
```



## milvus

### 安装

访问官网：https://milvus.io/

可以通过docker安装，非常方便，这里不展开说了，按照官网的Get Started一步一步操作就可以运行。

### 配置信息

- Host和Port

host和port不多说，milvus默认跑在19530端口。

- dataBase和collection

 milvus的结构和关系型数据库例如mysql、oracle有很大区别。

database作为一个库，创建database只需要指定一个名称即可。database下可以建很多的collection，每个collection的结构是固定的（各种字段），collection可以理解为mysql的表。

- dimension 维度

需要注意，这是向量的维度，这个值跟采用的向量算法有关。通常，文本和内容在被向量化后，会变成多个值来描述，这里有几个值就是有多少个维度。

例如我们用langchain4j包下的AllMiniLmL6V2EmbeddingModel向量模型来对一段话进行向量运算，得到的值是：

![image-20241029104611592](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/image-20241029104611592.png)

这里可以看到，我们得到的向量结果embedding1是一个长度为384的float数组，那么就表明这个向量模型会有384个维度。如果我们需要把这个向量结果保存到向量数据库中，建立collection的时候就需要指定向量字段的维度为384。

维度越高，可以携带的信息越多，能够描述更复杂的数据特征，例如图像、文本等，但同时，其计算量也会大幅上升。很多大语言模型都有向量计算的接口，能够返回上千维度的向量结果，这是十分可观的。

- metricType 距离度量类型

用于向量相似度计算。MetricType 指标决定了向量间距离的计算方式，从而影响相似向量的检索结果。常用的 MetricType 包括：

1. **L2 (Euclidean Distance)**：即欧几里得距离，用于测量两个向量之间的空间距离。数值越小，表示向量越接近。适合于密集型数据集和低维向量。
2. **IP (Inner Product)**：即内积距离，使用内积的方式来计算向量之间的相似度。适用于向量经过归一化后的余弦相似度计算，数值越大表示向量越接近。
3. **COSINE (Cosine Similarity)**：用于计算余弦相似度，即两个向量之间的夹角。通常先将向量归一化，再计算内积，因此和 IP 类似。
4. **HAMMING (Hamming Distance)**：哈希距离，用于稀疏向量和二进制向量。适用于通过 Locality-Sensitive Hashing (LSH) 的方式索引向量，特别是对于高维二进制向量。
5. **JACCARD**：杰卡德相似度，常用于离散集合，适合稀疏向量，用于量化两个集合之间的相似度。

不同的场景适用于不同的距离度量类型。选择合适的 `MetricType` 主要取决于你的数据特点、业务需求和所需的检索精度。以下是一些常见的使用场景和建议：

1. **密集型向量 (Dense Vectors)**

   - **L2 (Euclidean Distance)**：当向量表示空间位置、图像特征或一些数值型特征时，L2 是一个常见的选择，因为它能直接计算两个向量间的“距离”。L2 距离适合低维向量，且计算开销相对较低。
   - **Inner Product (IP)**：当你需要度量两个向量之间的相似度（如推荐系统）时，IP 可以通过计算内积来衡量相似性，尤其适合需要高召回率的场景。建议在使用前对向量进行归一化，以确保内积能够反映出实际的相似程度。
   - **Cosine Similarity**：如果关注两个向量的方向（而非大小），可以使用余弦相似度，尤其适合文本向量（如词嵌入）和高维密集向量。通常情况下，将向量归一化后计算 IP 可近似实现余弦相似度。

2. **稀疏型向量 (Sparse Vectors)**

   - **Hamming Distance**：当向量是高维二进制或稀疏向量时（例如通过 Locality-Sensitive Hashing (LSH) 生成的向量），哈希距离比较合适。Hamming 能有效地计算两个稀疏向量的差异，但一般不适合密集型数据。
   - **Jaccard Similarity**：当向量是离散的集合数据时，比如用户兴趣的集合表示，Jaccard 相似度适用。它常用于需要量化两个集合之间重叠程度的场景。
   - **Tanimoto Similarity**：与 Jaccard 相似度类似，但适用于有权重或频率特征的稀疏向量。可以有效处理非零项较多、集合间重叠量较大的情况。

3. **应用场景的实际需求**

   - **推荐系统**：如果你需要找到与用户特征最相似的物品，可以选择 `Inner Product` 或 `Cosine Similarity`。
   - **图像或音频特征检索**：通常 `L2` 或 `Cosine Similarity` 都适合，具体选哪一个还需根据数据的分布和具体效果调优。
   - **文本检索**：一般使用 `Cosine Similarity`，尤其是使用 BERT 或 Word2Vec 生成的文本向量时。

4. **资源和精度平衡**

   - 在一些高维度、需要实时检索的场景中，可以尝试一些较轻量级的距离度量（如 Hamming 或 IP）来平衡资源消耗和检索精度。
   - 使用稀疏向量时，通过选择合适的距离度量，如 Hamming 和 Jaccard，可以减少内存占用和加速检索速度。

综上所述，选择合适的 `MetricType` 需要根据具体业务、数据特点和资源限制进行综合评估和调试。

# 妙用

上面说到，使用embedding store可以保存文本的向量信息，再通过文本向量数据去查询，会根据向量的值给出分值（类似elasticsearch的逻辑）分值最高的数据是向量最接近的数据。而由于向量的特殊性，这样查询可以做到一种非常“智能”的感觉，远超过传统的检索结果。

如果说最基础的查询是mysql的like，进阶版是elasticsearch那样的分词，那么智能版的检索就是向量数据库了。下面看代码：

``` java
//定义一个向量存储
EmbeddingStore<TextSegment> embeddingStore = MilvusEmbeddingStore.builder()
    .host(MilvusConfig.getHost())
    .port(MilvusConfig.getPort())
    .databaseName(MilvusConfig.getDatabaseName())
    .collectionName("test")
    .dimension(384)
    .metricType(MetricType.L2)
    .build();
//定义一个向量模型
AllMiniLmL6V2EmbeddingModel embeddingModel = new AllMiniLmL6V2EmbeddingModel();
//文本内容
TextSegment segment1 = TextSegment.from("我喜欢踢足球");
//向量计算
Embedding embedding1 = embeddingModel.embed(segment1).content();
//往向量存储里保存
embeddingStore.add(embedding1, segment1);

//文本内容第二段
TextSegment segment2 = TextSegment.from("今天的天气真不错，非常晴朗。");
Embedding embedding2 = embeddingModel.embed(segment2).content();
embeddingStore.add(embedding2, segment2);

//线程等待，保证向量存储已经保存了
Thread.sleep(1000);

//需要查询的内容，同样经过向量运算
Embedding queryEmbedding = embeddingModel.embed("你喜欢什么运动？").content();
//从向量数据库中取回数据
List<EmbeddingMatch<TextSegment>> relevant = embeddingStore.findRelevant(queryEmbedding, 1);
//取回第一条
EmbeddingMatch<TextSegment> embeddingMatch = relevant.get(0);

//打印分值  0.8912628591060638  满分是1分
System.out.println(embeddingMatch.score());
//打印结果  我喜欢踢足球
System.out.println(embeddingMatch.embedded().text()); 
```

可以看到，这边输入的问题是：你喜欢什么运动，此时，向量数据库里存在两条信息：1.我喜欢踢足球  2.今天的天气真不错，非常晴朗。

如果此时是在mysql或者elasticsearch场景下，我们是无法检索信息的，查询的内容只会是Null。但是我们通过向量运算，不但能查询到内容，并且运动和足球有一个关联关系，能够把最贴近运动的内容找出来，这看起来就非常的“智能”了。





# 总结

Embedding在大语言模型应用中是非常重要的一个环节，这关系到内容的额外补充，可以作为最简单的训练来进行。常见的知识库能力就是基于Embedding来做的，而这一块拓展就可以有很大的想象空间和应用空间了。感谢您能够阅读到这儿，如果有什么问题欢迎在评论区沟通和交流，下期我们将认识一下AI应用最关键的RAG是什么东西。



PS:最近遇到了糟心事，需要维权。可能更新不会太快，或者会穿插更新维权经历，也算是丰富人生阅历了，从来没想过会遇到这种荒唐的事情，同样的也感受到普通人维权的各种困境。也许会有文章分享一下过程，再次感谢各位的阅读。

















