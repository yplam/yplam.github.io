---
title:  "不能忽略 .htaccess 对性能的影响"
categories: Linux Apache
---

在Apache环境下使用 .htaccess 进行URL重写或许已经是一种非常常用的方式；然而，如果URL重写规则比较复杂，使用.htaccess的方式可能会影响到Apache的性能。

**测试方式**

一个简单的echo语句，分别进行以下测试：

* allowoverride=none,无rewrite
* allowoverride=all,无.htaccess
* allowoverride=all,简单的.htaccess
* allowoverride=none,简单的rewrite
* allowoverride=all,复杂的.htaccess
* allowoverride=none,复杂的rewrite

**结果**

先说结果，当rewrite规则比较简单时，性能影响不大，17201 vs 19486。

当rewrite规则比较复杂，超过900行，有较多正则表达式，跳转（用来做栏目rewrite），性能变化较大 1497 vs 7108

如果访问的目录路径比较深，在allowoverride开启的情况下，apache需要历遍每一个目录去读取.htaccess，性能的影响理论上会更大，测试数据 15623 vs 17201（7层目录vs根目录）。

当然，在实际应用场景中，相对PHP跟Mysql而言，Apache的这点性能损失可能不算什么；但是，只需要简单的将rewrite规则从htaccess复制到apache.conf中就可以得到性能的提升，何乐而不为呢？ :)



**测试数据**

allowoverride=none，无rewrite

```
Concurrency Level:      10
Time taken for tests:   0.500 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    20008.88 [#/sec] (mean)
Time per request:       0.500 [ms] (mean)
Time per request:       0.050 [ms] (mean, across all concurrent requests)
Transfer rate:          3673.51 [Kbytes/sec] received
```

allowoverride=all，无.htaccess

```
Concurrency Level:      10
Time taken for tests:   0.491 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    20371.66 [#/sec] (mean)
Time per request:       0.491 [ms] (mean)
Time per request:       0.049 [ms] (mean, across all concurrent requests)
Transfer rate:          3740.11 [Kbytes/sec] received
```

简单的.htaccess(符合单入口文件类的网站)

```
Concurrency Level:      10
Time taken for tests:   0.581 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    17201.99 [#/sec] (mean)
Time per request:       0.581 [ms] (mean)
Time per request:       0.058 [ms] (mean, across all concurrent requests)
Transfer rate:          3158.18 [Kbytes/sec] received
```


简单.htaccess，访问 /a/b/c/d/e/f/g/ 路径

```
Concurrency Level:      10
Time taken for tests:   0.640 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    15623.29 [#/sec] (mean)
Time per request:       0.640 [ms] (mean)
Time per request:       0.064 [ms] (mean, across all concurrent requests)
Transfer rate:          2868.34 [Kbytes/sec] received

```

简单的rewrite规则，写入apache.conf

```
Concurrency Level:      10
Time taken for tests:   0.513 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    19486.15 [#/sec] (mean)
Time per request:       0.513 [ms] (mean)
Time per request:       0.051 [ms] (mean, across all concurrent requests)
Transfer rate:          3577.54 [Kbytes/sec] received
```

复杂的.htaccess(900行，使用正则表达式对栏目内容进行映射，还有对旧内容的重定向等)

```
Concurrency Level:      10
Time taken for tests:   6.677 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    1497.75 [#/sec] (mean)
Time per request:       6.677 [ms] (mean)
Time per request:       0.668 [ms] (mean, across all concurrent requests)
Transfer rate:          274.98 [Kbytes/sec] received

```

将规则写入apache.conf，并且禁用allowoverride后：

```
Concurrency Level:      10
Time taken for tests:   1.407 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1880000 bytes
HTML transferred:       10000 bytes
Requests per second:    7108.94 [#/sec] (mean)
Time per request:       1.407 [ms] (mean)
Time per request:       0.141 [ms] (mean, across all concurrent requests)
Transfer rate:          1305.16 [Kbytes/sec] received

```
