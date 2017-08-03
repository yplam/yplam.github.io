---
title:  "Netflix Prize 相关资料"
categories: MachineLearning
---

Netflix Prize 是2006年Netflix启动的一个机器学习和数据挖掘比赛，旨在解决电影评分预测问题。

训练集包含 480,189 名用户对 17,770 部电影的 100,480,507 份评分，评分为 1-5 分。

qualifying.txt 中包含需要检验的数据，参赛者根据检验集提供的电影id跟用户id，时间进行评分预测，然后提交结果。（不过因为没有关于结果评分的数据，所以这个文件貌似用处不大）

probe.txt 相当于测试集，与 qualifying.txt 的区别在于你可以在训练集中查到对应的评分数据。

结果的评估使用 RMSE。


**数据集下载：**

* [百度网盘](http://pan.baidu.com/s/1hAdpU)
* [Netflix Prize data](https://www.kaggle.com/netflix-inc/netflix-prize-data)

**论文:**

* [Yehuda Koren,The BellKor Solution to the Netflix Grand Prize](http://www.valleytalk.org/wp-content/uploads/2014/04/1GrandPrize2009_BPC_BellKor.pdf)
* [A. Töscher, M. Jahrer, R. Bell,The BigChaos Solution to the Netflix Grand Prize](http://www.valleytalk.org/wp-content/uploads/2014/04/2GrandPrize2009_BPC_BigChaos.pdf)
* [M. Piotte, M. Chabbert,The Pragmatic Theory solution to the Netflix Grand Prize ](http://www.valleytalk.org/wp-content/uploads/2014/04/3GrandPrize2009_BPC_PragmaticTheory.pdf)

**相关文章**

* [Netflix Prize and SVD](http://buzzard.ups.edu/courses/2014spring/420projects/math420-UPS-spring-2014-gower-netflix-SVD.pdf) 这篇文章对获奖算法做了比较详细的讲解
* [Predicting movie ratings and recommender systems](http://arek-paterek.com/book/)一份关于Netflix Prize的195页专刊
* [关于Netflix Prize的总结(翻译)](http://blog.csdn.net/songzitea/article/details/42024399)
* [Winning the Netflix Prize: A Summary](http://blog.echen.me/2011/10/24/winning-the-netflix-prize-a-summary/)
* [Netflix推荐系统：从评分预测到消费者法则](http://blog.csdn.net/lzt1983/article/details/7696578)

**参考资料**

* [【Netflix】Netflix Prize 竞赛数据集](http://lazynight.me/3294.html)
* [Netflix推荐算法](http://www.valleytalk.org/2014/04/28/netflix%E6%8E%A8%E8%8D%90%E7%AE%97%E6%B3%95/)

**关于Netflix的一些报道：**

* [Netflix：从传统DVD租赁向流媒体华丽转身](http://stock.hexun.com/2011-05-11/129510510.html)
* [Netflix 是如何用大數據捧紅「紙牌屋」的](https://www.inside.com.tw/2013/02/26/netflix-big-data-houseof-card)

