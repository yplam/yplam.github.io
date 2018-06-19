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

然后创建 Elasticsearch 表索引，最后将 RDD 保存到 Elasticsearch，我们可以先看看数据在 Elasticsearch 中的存在方式，这样会更加清晰：

```
GET /urindex_1529048147127/_search
```

```
  {
    "_index": "urindex_1529048147127",
    "_type": "items",
    "_id": "Iphone 4",
    "_score": 1,
    "_source": {
      "defaultRank": 5,
      "expires": "2018-06-18T10:45:41.677+08:00",
      "countries": [
        "United States",
        "Canada",
        "Estados Unidos Mexicanos"
      ],
      "id": "Iphone 4",
      "date": "2018-06-16T10:45:41.677+08:00",
      "category-pref": [
        "tablets"
      ],
      "categories": [
        "Phones",
        "Electronics",
        "Apple"
      ],
      "available": "2018-06-14T10:45:41.677+08:00",
      "purchase": [
        "Ipad-retina",
        "Iphone 6"
      ],
      "popRank": 4,
      "view": [
        "Soap",
        "Tablets"
      ]
    }
  }
```

很明显这是一个基于产品的推荐系统，主事件为 purchase ，所以直观的解释就是 “购买了 Iphone 4 的用户还购买了 Ipad-retina，Iphone 6；购买了 Iphone 4 的用户浏览了 Soap，Tablets；Iphone 4 属于 Phones，Electronics，Apple 分类……”

UR 模型的 predict 过程实际就是根据查询输入以及用户的历史事件（从Event Storage中查出），构造 ES 的 json query。

模型定义查询结构如下：

```

case class Query(
  user: Option[String] = None, // must be a user or item id
  userBias: Option[Float] = None, // default: whatever is in algorithm params or 1
  item: Option[String] = None, // must be a user or item id
  itemBias: Option[Float] = None, // default: whatever is in algorithm params or 1
  itemSet: Option[List[String]] = None, // item-set query, shpping cart for instance.
  itemSetBias: Option[Float] = None, // default: whatever is in algorithm params or 1
  fields: Option[List[Field]] = None, // default: whatever is in algorithm params or None
  currentDate: Option[String] = None, // if used will override dateRange filter, currentDate must lie between the item's
  // expireDateName value and availableDateName value, all are ISO 8601 dates
  dateRange: Option[DateRange] = None, // optional before and after filter applied to a date field
  blacklistItems: Option[List[String]] = None, // default: whatever is in algorithm params or None
  returnSelf: Option[Boolean] = None, // means for an item query should the item itself be returned, defaults
  // to what is in the algorithm params or false
  num: Option[Int] = None, // default: whatever is in algorithm params, which itself has a default--probably 20
  from: Option[Int] = None, // paginate from this position return "num"
  eventNames: Option[List[String]], // names used to ID all user actions
  withRanks: Option[Boolean] = None) // Add to ItemScore rank fields values, default false
    extends Serializable
    
```

然后转换为类似下面的请求：

```
  {
    "size": 20
    "query": {
      "bool": {
        "should": [
          {
            "terms": {
              "rate": ["0", "67", "4"]
            }
          },
          {
            "terms": {
              "buy": ["0", "32"],
              "boost": 2
            }
          },
          { // categorical boosts
            "terms": {
              "category": ["cat1"],
              "boost": 1.05
            }
          }
        ],
        "must": [ // categorical filters
          {
            "terms": {
              "category": ["cat1"],
              "boost": 0
            }
          },
         {
        "must_not": [//blacklisted items
          {
            "ids": {
              "values": ["items-id1", "item-id2", ...]
            }
          },
         {
           "constant_score": {// date in query must fall between the expire and available dates of an item
             "filter": {
               "range": {
                 "availabledate": {
                   "lte": "2015-08-30T12:24:41-07:00"
                 }
               }
             },
             "boost": 0
           }
         },
         {
           "constant_score": {// date range filter in query must be between these item property values
             "filter": {
               "range" : {
                 "expiredate" : {
                   "gte": "2015-08-15T11:28:45.114-07:00"
                   "lt": "2015-08-20T11:28:45.114-07:00"
                 }
               }
             }, "boost": 0
           }
         },
         {
           "constant_score": { // this orders popular items for backfill
              "filter": {
                 "match_all": {}
              },
              "boost": 0.000001 // must have as least a small number to be boostable
           }
        }
      }
    }
  }
```

Universal Recommender 虽然原理比较简单，不过它巧妙地利用 Elasticsearch 将业务规则整合进来的方式确实可以给人带来启发。

以上仅为个人总结笔记，能力所限可能有比较多错误，欢迎交流指正 yplam(at)yplam.com

