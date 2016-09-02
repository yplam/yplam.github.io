---
title:  "30个最常用的Symfony组件"
categories: PHP Symfony
---

在Symfony官网上有个投票，让用户选出工作中最有用的组件，下面是评选结果：

1. [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle) (60%) 提供用户注册、管理相关的功能，推荐！
2. [FOSRestBundle](https://github.com/FriendsOfSymfony/FOSRestBundle) (30%) Restful Api，推荐！
3. [KnpMenuBundle](https://github.com/KnpLabs/KnpMenuBundle) (25%) 菜单功能，推荐！
4. [StofDoctrineExtensionsBundle](https://github.com/stof/StofDoctrineExtensionsBundle) (25%) 提供一堆有用的的Doctrine扩展，推荐！
5. [JMSSerializerBundle](https://github.com/schmittjoh/JMSSerializerBundle) (24%) 序列化支持，可以与FOSRestBundle一起使用
6. [SonataAdminBundle](https://github.com/sonata-project/SonataAdminBundle) (24%) 非常好用的一个管理后台组件，极力推荐！
7. [FOSJsRoutingBundle](https://github.com/FriendsOfSymfony/FOSJsRoutingBundle) (23%) 将路由信息开放给前端
8. [LiipImagineBundle](https://github.com/liip/LiipImagineBundle) (22%) 类似Drupal中的Imagecache功能，为图片提供自动的缩放切割功能，推荐！
9. [KnpPaginatorBundle](https://github.com/KnpLabs/KnpPaginatorBundle) (16%) 提供分页功能，推荐！
10. [NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle) (11%) 在代码注释中提供文档，可以在线阅读
11. [FOSElasticaBundle](https://github.com/FriendsOfSymfony/FOSElasticaBundle) (11%) 整合  ElasticSearch
12. [HWIOAuthBundle](https://github.com/hwi/HWIOAuthBundle) (11%) 提供OAuth登陆功能，国内的新浪、qq等都有提供，并且容易扩展，极力推荐！
13. [VichUploaderBundle](https://github.com/dustin10/VichUploaderBundle) (10%) 自动帮你上传文件
14. [RaulFraileLadyBugBundle](https://github.com/raulfraile/LadybugBundle) (9%) 输出调试信息，替代var_dump，类似Drupal的dsm
15. [KnpSnappyBundle](https://github.com/KnpLabs/KnpSnappyBundle) (8%) 生成pdf或者图片
16. [DoctrineFixturesBundle](https://github.com/doctrine/DoctrineFixturesBundle) (8%) Doctrine2 Data Fixtures library. into Symfony so that you can load data fixtures programmatically into the Doctrine ORM or ODM.
17. [WhiteOctoberPagerFantaBundle](https://github.com/whiteoctober/WhiteOctoberPagerfantaBundle) (8%) 分页相关功能
18. [DoctrineMigrationsBundle](https://github.com/doctrine/DoctrineMigrationsBundle) (8%) 提供Doctrine升级数据库相关的功能
19. [MopaBootstrapBundle](https://github.com/phiamo/MopaBootstrapBundle) (7%) Bootstrap模板相关
20. [JMSSecurityExtraBundle](https://github.com/schmittjoh/JMSSecurityExtraBundle) (6%) 扩展Symfony的Securety
21. [DoctrineBundle](https://github.com/doctrine/DoctrineBundle) (5%) Doctrine
22. [AvalancheImagineBundle](https://github.com/avalanche123/AvalancheImagineBundle) (5%) 类Imagecache功能，已经不维护
23. [JMSDiExtraBundle](https://github.com/schmittjoh/JMSDiExtraBundle) (5%) 扩展Symfony的依赖注入
24. [GenemuFormBundle](https://github.com/genemu/GenemuFormBundle) (4%) 提供一些常用的表单
25. [KnpGaufretteBundle](https://github.com/KnpLabs/KnpGaufretteBundle) (4%) 文件管理
26. [SncRedisBundle](https://github.com/snc/SncRedisBundle) (4%) Redis
27. [LexikFormFilterBundle](https://github.com/lexik/LexikFormFilterBundle) (3%) This Symfony2 bundle aims to provide classes to build a form filter and then build a doctrine query from this form filter.
28. [JMSI18nRoutingBundle](https://github.com/schmittjoh/JMSI18nRoutingBundle) (3%) This bundle allows you to create i18n routes.
29. [LiuggioExcelBundle](https://github.com/liuggio/ExcelBundle) (3%) This bundle permits you to create, modify and read excel objects.
30. [JMSTranslationBundle](https://github.com/schmittjoh/JMSTranslationBundle) (3%) This bundle puts the Symfony Translation Component on steroids.

**总结：**

作为一名从Drupal转过来的Symfony新手，感觉这个投票结果还是挺熟悉的。其中前10名中有9个组件曾经在自己的项目中用过（除了FOSJsRoutingBundle，可能是因为自己做的都是中规中矩的网站）。

其中，FOSUserBundle + HWIOAuthBundle + SonataAdminBundle 基本上是项目开展时的标配，提供用户与后台

KnpMenuBundle、 LiipImagineBundle、 KnpPaginatorBundle、StofDoctrineExtensionsBundle 基本上也是必备模块

配置好上面这些模块，会发现：咦？一个熟悉的Drupal基础框架就出来了，有了用户、后台、pager、imagecache、菜单，下面就可以完全按你的想法去开发，Cool！

PS：其实这些功能Drupal基本上都是已经自带有，但虽然使用Drupal可能可以在一两天内搭出一个满足你90%需求的网站（当然，不包括主题），但剩下的10%会成为你漫长的折磨 :)
