---
title:  "Symfony 4 尝鲜"
categories: PHP Symfony
---

Symfony 4 如期在11月的最后一天发布，相比与 Symfony 2 到 Symfony 3 的升级，Symfony 4 的变化似乎更大，特别是由于加入 Flex 以及 Service Autowired，开发方式都会发生比较大的变化。

Symfony 4 主要特性：

* 基于 Symfony Flex 的包自动化安装。
* Service 的 Auto-registered 与 autowired，根据代码自动生成，无需配置。
* Symfony 4 成了一个开箱即用的微框架，比 Symfony 3 小 70%。
* 性能优化
* MakerBundle、Webpack 等新特性

安装：

```

composer create-project symfony/skeleton my_project
 
```

安装后的第一感觉变化还是蛮大，web 变到 public，里面就只有一个 index.php 文件，通过 $_SERVER['APP_ENV'] 或者 .env 文件来加载环境参数。

目录结构也有很大的变化，而且都变得很精简。

但这个精简版的 Symfony 4 功能还有点弱，至少不支持常用的 annotations、twig 以及 profiler，不过得益于Flex的存在，安装配置也简单：

```

composer require annotations
composer require twig
composer require --dev profiler

```

Flex 会根据需要将默认配置文件下到本地。


