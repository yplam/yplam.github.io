---
title:  "Discuz移动接口原理简介"
categories: PHP Discuz
---

Discuz论坛内置提供针对移动端开发的json接口，下面根据源码进行一些分析。

**入口 api/mobile/index.php**

功能比较简单，指向source/plugins/mobile/mobile.php 或者check.php，check.php只有当类似$_GET['check'] == 'check'的情况下才运行，返回系统基本信息；其他请求均通过mobile.php接管。

**mobile.php**

mobile.php的基本思想是根据$_GET['module']以及$_GET['version']来调用相关的module文件，$_GET['version']参数可以做到版本兼容。

**mobile/api/%version%/%module%.php**

实现相当简单，根据需求设置$_GET['mod']等参数，然后将请求转到Discuz网页版页面，完成请求。同时，文件中定义mobile_api类。

**mobile_api类**

mobile_api类包含两个方法:common与output，common方法对应discuz的runhooks调用；output方法对应discuz的hookscriptoutput调用。

**流程**

Discuz调用runhooks时，调用mobile_plugin->common()，调用mobile_api->common()

Discuz在模板中调用hookscriptoutput()时，调用mobile_plugin->global_mobile()，调用mobile_api->output()

**总结**

Discuz大量使用$_G全局变量来保存系统运行过程中的结果，这为移动端内容输出提供便利，只需要在模板输出前截获$_G变量，然后根据需求从$_G变量中提取所需数据，并使用json格式返回。这就是Discuz移动端接口的基本实现方式。
