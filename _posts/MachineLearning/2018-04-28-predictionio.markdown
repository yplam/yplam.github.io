---
title:  "PredictionIO实现论坛广告贴过滤"
categories: MachineLearning PredictionIO
published: true
---

关注 Apache PredictionIO 已有一段时间，最近需要做一个过滤论坛广告贴的功能，调研了一番市面上提供文本检测的API服务，发现靠谱的公司都比较贵，并且准确率不高（一个可能性是他们使用的训练数据集是互联网上的通用信息，而我们是一个垂直行业论坛，攻击来源也可能是固定的那么若干个），于是尝试自己实现个文本分类器来过滤广告。

关于文本分类，首先想到的就是朴素贝叶斯分类器，其理论清晰简单，并且各大机器学习框架均支持。

**朴素贝叶斯分类器** 关于贝叶斯分类器的的原理可以参考以下文档：[算法杂货铺——分类算法之朴素贝叶斯分类(Naive Bayesian classification)](http://www.cnblogs.com/leoo2sk/archive/2010/09/17/naive-bayesian-classifier.html)

### Spark实现

下面是 Spark 贝叶斯分类器的实现，主要流程为:

1. 读入数据集
2. 使用 ansj 分词，生成 (label, words) 结构的 dataframe
3. 分割训练集与测试集
4. 创建 HashingTF，IDF，NaiveBayes 的 Pipeline 模型
5. 设置参数矩阵，然后使用 TrainValidationSplit 进行模型训练与参数选择
6. 训练出 bestModal，保存到指定路径
7. 通过最佳模型与测试集得出模型评估

实现代码如下：

```

package com.yplam

import org.apache.log4j.{Level, Logger}
import org.apache.spark.sql.{Row, SparkSession}
import scopt.OptionParser
import org.ansj.recognition.impl.StopRecognition
import org.ansj.splitWord.analysis.ToAnalysis
import org.apache.spark.ml.{Pipeline, PipelineModel}
import org.apache.spark.ml.classification.NaiveBayes
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator
import org.apache.spark.ml.feature.{HashingTF, IDF}
import org.apache.spark.ml.tuning.{ParamGridBuilder, TrainValidationSplit}
import org.apache.spark.mllib.evaluation.MulticlassMetrics

object SpamFilterTrainer {

  /**
    * 运行参数
    * @param trainInput 训练数据集路径，格式为 label;content
    */
  case class SpamFilterParams(
               trainInput: String = null,
               modelOutput: String = null
             )

  def main(args: Array[String]): Unit = {
    Logger.getLogger("org").setLevel(Level.WARN)
    val spark = SparkSession
      .builder()
      .appName(this.getClass.getSimpleName)
      .master("local[*]")
      .getOrCreate()

    val defaultParam = SpamFilterParams()
    val parser = new OptionParser[SpamFilterParams](this.getClass.getSimpleName) {
      head(s"${this.getClass.getSimpleName}: BBS Spam filter.")
      opt[String]("train")
        .text("train input")
        .action((x, c) => c.copy(trainInput = x))
        .required()
      opt[String]("output")
        .text("model output")
        .action((x, c) => c.copy(modelOutput = x))
        .optional()
    }
    parser.parse(args, defaultParam).map { params =>
      run(spark, params.trainInput, params.modelOutput)
      sys.exit(0)
    } getOrElse {
      sys.exit(1)
    }
  }

  def run(spark: SparkSession, trainInput: String, modelOutput: String) : Unit = {
    import spark.implicits._
    import scala.collection.JavaConversions._

    val logger = Logger.getLogger("app")
    logger.warn("Start:" + System.currentTimeMillis().toString)

    val filter = new StopRecognition()
    filter.insertStopWords(" ", ",")

    // 读取数据集，使用 ansj 分词，生成 (label, words) 结构的 dataframe
    val inputData = spark.sparkContext.textFile(trainInput).map{ line =>
      val splits = line.split(";", 2)
      val content = if(splits(1).length > 0){
        ToAnalysis.parse(splits(1)).recognition(filter).getTerms.map(_.getRealName)
      }
      else List("")
      (splits(0).toInt, content)
    }.toDF("label", "words")
    inputData.show()

    // 分割训练集与测试集
    val Array(train, test) = inputData.randomSplit(Array(0.9, 0.1), seed = 12345).map(_.cache())

    // 创建 HashingTF，IDF，NaiveBayes 的 Pipeline 模型
    val hashingTF = new HashingTF().setInputCol("words").setOutputCol("rawFeatures").setNumFeatures(5000)
    val idf = new IDF().setInputCol("rawFeatures").setOutputCol("features")
    val nb = new NaiveBayes()
      .setModelType("multinomial")
      .setFeaturesCol("features")
      .setLabelCol("label")
    val pipeline = new Pipeline()
      .setStages(Array(hashingTF, idf, nb))

    // 设置参数矩阵，然后使用 TrainValidationSplit 进行模型训练与参数选择
    val paramGrid = new ParamGridBuilder()
      .addGrid(hashingTF.numFeatures, Array(50000, 100000))
      .addGrid(nb.smoothing, Array(1.0))
      .build()

    val trainValidationSplit = new TrainValidationSplit()
      .setEstimator(pipeline)
      .setEvaluator(new MulticlassClassificationEvaluator)
      .setEstimatorParamMaps(paramGrid)
      .setTrainRatio(0.8)


    val model = trainValidationSplit.fit(train)

    logger.warn("Train end:" + System.currentTimeMillis().toString)

    // 训练出 bestModal，保存到指定路径
    if(!modelOutput.isEmpty)
      model.bestModel.asInstanceOf[PipelineModel].write.overwrite().save(modelOutput)

    // 将参数与评估结果打印出来
    model.getEstimatorParamMaps.zip(model.validationMetrics).foreach(println)

    val testResult = model.transform(test)
      .select("features", "label", "prediction")
    testResult.show()
    val testPredictionAndLabels = testResult.select("prediction", "label").rdd.map{
      case Row(prediction: Double, label: Int) => (prediction, label.toDouble)
    }
    val metrics = new MulticlassMetrics(testPredictionAndLabels)
    println(metrics.confusionMatrix)
    println("accuracy:" + metrics.accuracy)
    println("weightedPrecision:" + metrics.weightedPrecision)
    println("weightedRecall:" + metrics.weightedRecall)

    logger.warn("Finish:" + System.currentTimeMillis().toString)

    spark.stop()
  }
}


```

打包，然后在 Spark 中训练：

```

spark-submit --class com.yplam.SpamFilterTrainer spark-scala-playground.jar --train /home/yplam/spark/work/data/bbs --output /home/yplam/spark/work/model/bbs

```

结果还算可以，约 93% 的准确率。由于我们之前已经保存好最佳模型，可以编写一个简单的json API接口对外提供服务，而另一个可能更加适合的方式就是将模型移到 PredictionIO 上。

### PredictionIO 简介

[PredictionIO](https://predictionio.apache.org) 是一个开源的机器学习平台，用于构建、评估与发布机器学习算法引擎。PredictionIO 提供事件服务器( Event Server ) 用于接收与保存数据，用于生成训练数据集；引擎模板( engine template )提供各种不同场景应用所需的机器学习算法模板。模型训练出来后 PredictionIO 提供保存与发布服务，发布后的模型可以通过接收 REST API 提供预测结果。对开发者而言，只需要按照 PredictionIO 提供的 DASE 模式实现数据源与预处理(D)、算法(A)、服务(S)、评估指标(E)。PredictionIO 使开发者专注于模型本身，其他的事情它都已经帮你实现。

![PredictionIO]({{ site.url }}/assets/2018/predictionio-intro.png "PredictionIO")

上面为 PredictionIO 的单模型引擎框架图。

事件服务器接收与存储事件数据，数据可以有多种格式，譬如对于文本分类器可以是 { "entityId" : 123, "entityType" : "content", "event" : "$set", "properties" : { "label" : "spam", "text" : "哈哈哈"} }，对于推荐引擎可以是 { "entityId" : 1, "entityType" : "user", "event" : "buy", "targetEntityType":"item", "targetEntityId":"123" } 等。关于事件模型的更多内容可以参考 [Events Modeling](https://predictionio.apache.org/datacollection/eventmodel/) 。 PredictionIO 提供 REST 接口用于接收事件 多种存储后端，包括mysql，pgsql 跟 hbase。


### PredictionIO 实现

由于 PredictionIO 社区提供了大量的引擎模板，因此实现自己模型的最简单方式就是在官方模板的基础上进行修改，譬如 [Text Classification](https://github.com/apache/predictionio-template-text-classifier)中就包含基于贝叶斯算法的文本分类器实现，得益于 PredictionIO 的 DASE 模型，我们只需要拷贝此项目，根据项目实际情况做小修改，然后加入 ansj 分词功能，完成。

针对 DataSource，我们需要根据我们模型实际数据结构，设定 entityType，eventNames，以及对 Observation 结构的处理。

针对 Preparator，我们用 ansj 替换原有的分词方式，同时限制了内容长度。

相关代码地址：https://github.com/yplam/predictionio-spam-detection-cn

后记：通过分析“漏网之鱼”的内容，发现已经有针对贝叶斯算法的攻击，也就是在长篇内容中夹入少量垃圾信息，通过只截取标题+内容头尾各500个字的方式，可以有一定的效果提示。





