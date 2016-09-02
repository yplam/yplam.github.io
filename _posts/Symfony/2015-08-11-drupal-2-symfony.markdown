---
title:  "如何将一个Drupal网站转到Symfony"
categories: PHP Symfony Drupal
---

最近尝试将自己的一些Drupal网站转移到Symfony（这个个人博客就是其中之一），下面分享一些转移过程的心得，主要包括Drupal基本功能在Symfony上的实现。

**用户管理**

FOSUserBundle提供用户登录、注册、找回密码等基本功能，并且提供个可选的Group功能，将用户分组，按组设定权限。

**权限管理**

* 对于常规路径的权限（类似Drupal的HookMenu），Symfony 可以在security中根据角色权限配置，结合FOSUserBundle的分组功能可以容易搞掂。
* 对于某个特定内容的CRUD权限，Symfony可以用[Voter](http://symfony.com/doc/current/cookbook/security/voters_data_permission.html)的方式来设定

**管理后台**

使用SonataAdminBundle，只需要简单的编写一个服务，就可以完成常规的CRUD后台，对于特定需求，可以通过extend CRUD Controller的方式完成，而针对非CRUD类的管理后台，可以[使用自定义页面](http://yplam.com/post/69)。

**Hook钩子**

Drupal中钩子可以说是最基础的功能，而Symfony中可以很方便的用Event来实现。

**数据迁移**

* 对于简单型网站（如这个博客），可以使用 [node_export](https://www.drupal.org/project/node_export) 模块导出XML文件，然后在Symfony下写个Command导入内容。

* 用户或者内容结构较为复杂的网站，可以直接写Command，通过分析数据库结构用SQL语句导入内容。

* 由于Symfony与Drupal的密码加密方式不一致，我们可以使用自己的PasswordEncoder服务，覆盖 isPasswordValid 方法，对旧密码加密方式做兼容。

* 对path的兼容比较麻烦，在此的处理方式是使用一个controller专门做旧内容的跳转。当然，也可以通过监听 NotFoundHttpException 事件来进行处理跳转

**比Drupal更好用的功能**

首先，最明显的就是网站速度变快了，并且Symfony可以有多种方式来管理缓存，优化性能，而Drupal针对登录用户的缓存功能相对较弱。

针对开发者而言，网站deploy变得非常简单，使用capifony以及doctrine migrations，一条命令可以搞掂。

网站模板不在是一件痛苦的事情，使用twig基本上可以完美的将一个html模板重现；而Drupal，可能70%以上的时间需要花在模板上。

更新：现在网站已迁移到github。
