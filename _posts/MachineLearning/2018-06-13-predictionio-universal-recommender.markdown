---
title:  "PredictionIO Universal Recommender 笔记"
categories: MachineLearning PredictionIO
published: true
---

Universal Recommender (UR) 是 ActionML 公司基于 PredictionIO 开发的协同过滤引擎，使用 Correlated Cross-Occurrence (CCO) 算法，可以应用与个性化推荐、物品相关推荐、购物车推荐、基于业务逻辑的推荐等。

COO算法是协同过滤算法的一种，然而一般的协同过滤只能针对单一行为，CCO算法可以计算交叉行为下的协同关联。例如：它不仅可以通过用户的浏览行为来告诉你 “浏览了内容A的人可能会浏览内容B” ,它还能结合用户的浏览行为和用户的广告点击行为来告诉你 “点击了广告A的人可能会浏览内容F”。关于COO的详细介绍可参考下面文章：[基于 CCO 的协同过滤推荐](http://hejunhao.me/archives/1083)，[Multi-domain predictive AI or how to make one thing predict another](https://developer.ibm.com/dwblog/2017/mahout-spark-correlated-cross-occurences/)。


**PredictionIO 与 UR 安装**

参考 PredictionIO 与 UR 的官方文档安装，需要注意的是 UR 需要使用 Elasticsearch 来保存结果，现时（PredictionIO 0.12.1）默认支持的 Elasticsearch 版本为 5.*。最简单的方式就是下载tar包然后解压到 PredictionIO 的 vonders/elasticsearch-5.*.* 目录，然后在pio-env.sh文件进行配置。

配置完成后可以在 ur 的目录下运行以下命令进行测试：

```
./examples/integration-test
```

此命令会使用 examples/handmade-engine.json 的配置，data/sample-handmade-data.txt 中的数据，导入数据，build模型，deploy模型，使用预设数据进行测试并且与预设结果进行对比。

**DASE实现**

DataSource 只接受 user-item 类型，事件类型可以有多种，如测试数据中的 purchase、view、category-pref，其中第一个为主事件，其余为次要事件。可以使用 $set 请求定义 item 属性。DataSource 读取事件日志，按事件名聚合事件；读取 item 信息，并且将 Properties 信息附加到 item 上；然后返回事件、Item相关的RDD。

Preparator 根据 minEventsPerUser 的配置过滤掉主事件过少的用户，然后过滤掉没有主事件的用户以及其次要事件。因为 UR 算法需要用到 mahout，Preparator 需要将事件 RDD 转换为 org.apache.mahout.sparkbindings.indexeddataset.IndexedDatasetSpark 供算法使用。IndexedDatasetSpark 包含 userID 的 BiDictionary，itemID 的 BiDictionary 以及 经过 BiDictionary key转换的 user-item 矩阵。

Serving 与 RecommendationEngine 分别为用户请求的响应处理以及引擎的构造描述，较常规没有太多特殊内容。

URAlgorithm 根据 URAlgorithmParams 配置训练模型，保存，然后做预测。train 过程计算主事件矩阵与自身以及个次要事件的 Log Likelihood Ratio（LLR） 矩阵，然后保存到 ElasticSearch。predict 过程将用户的输入转换为对 ElasticSearch 的搜索请求，此为模型核心。

模型参数保存：历遍各 LLR 矩阵中的每一行，按降序排序每行的 releate_item，并且构造成 (itemID, Map(actionName -> JArray(vector[releate_item]))) ，然后通过 groupAll 操作，将每个 item 的事件内容与属性内容集合到一起，每个条目均得到类似以下格式内容：

```
(Ipad-retina,Map(defaultRank -> JDouble(3.0), expires -> JString(2018-06-15T17:09:41.677740+00:00), countries -> JArray(List(JString(United States), JString(Estados Unidos Mexicanos))), date -> JString(2018-06-13T17:09:41.677740+00:00), category-pref -> JArray(List(JString(tablets))), categories -> JArray(List(JString(Tablets), JString(Electronics), JString(Apple))), available -> JString(2018-06-11T17:09:41.677740+00:00), purchase -> JArray(List(JString(Iphone 6), JString(Iphone 4))), popRank -> JDouble(2.0), view -> JArray(List(JString(Soap)))))

```

然后创建 Elasticsearch 表索引，然后将 RDD 保存到 Elasticsearch



