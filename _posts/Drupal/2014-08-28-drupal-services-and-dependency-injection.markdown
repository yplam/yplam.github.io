---
title:  "Drupal 8服务与依赖注入"
categories: PHP Drupal
---

原文地址：[Services and dependency injection in Drupal 8](https://www.drupal.org/node/2133171)

Drupal 8 引入了服务的概念，用来解耦可复用的功能，并且可以通过在服务容器中注册这些服务，让它们可插拔与可替换。身为开发者，最佳的方式就是通过服务容器来访问Drupal提供的所有服务，这样可以保证遵循系统的解耦特性。在[Symfony 2的文档](http://symfony.com/doc/current/book/service_container.html)中对服务有非常好的介绍。

对开发者而言，服务用来执行类似访问数据库，发送邮件等操作。我们不使用PHP原生的MySQL函数，而是通过服务容器来使用Drupal提供的核心服务来执行这些操作，这样我们的代码可以很简单的访问数据库，而不需要考虑数据库是MySQL还是SQLite，同样，发邮件时也不需要考虑是通过SMTP还是其他方式。

**核心服务**

核心服务在[CoreServiceProvider.php](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21CoreServiceProvider.php/8) 与 [core.services.yml ](https://api.drupal.org/api/drupal/core%21core.services.yml/8)中定义。如：

{% highlight yml %}
      ...
    language_manager:
        class: Drupal\Core\Language\LanguageManager
        arguments: ['@language.default']
      ...
      path.alias_manager:
        class: Drupal\Core\Path\AliasManager
        arguments: ['@path.crud', '@path.alias_whitelist', '@language_manager']
      ...
      string_translation:
        class: Drupal\Core\StringTranslation\TranslationManager
      ...
      breadcrumb:
        class: Drupal\Core\Breadcrumb\BreadcrumbManager
        arguments: ['@module_handler']
      ...
{% endhighlight %}

某个服务可以依赖于其他服务。譬如在上面的例子中，path.alias_manager在arguments参数中指定依赖服务 path.crud， path.alias_whitelist 与 language_manager。可以通过@服务名的方式来定义一个服务的依赖，如@language_manager。当Drupal中的某代码请求path.alias_manager服务时，服务容器会保证path.crud， path.alias_whitelist 与 language_manager 服务可以传递给path.alias_manager的构造函数，因此它会先请求上面的每一个服务。依次地，language_manager依赖于language.default，等等等。

Drupal包含大量的服务，获取该列表的最好方法是查看[CoreServiceProvider.php](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21CoreServiceProvider.php/8) 与 [core.services.yml ](https://api.drupal.org/api/drupal/core%21core.services.yml/8) 文件。

服务容器（或者叫依赖注入容器）是一个管理服务实例的PHP对象。Drupal的服务容器是在Symfony服务容器的基础上建立的，关于文件结构、特殊字符、可选依赖等相关内容文档可以查看[Symfony 2服务容器文档](http://symfony.com/doc/current/book/service_container.html)。

**通过依赖注入来访问服务**

在Drupal 8中，依赖注入是访问服务的首选方式，并且应该尽可能的使用。服务通过构造函数参数或者setter方法的方式进行传递，而不是直接调用全局的服务容器。很多由核心模块提供的控制器以及插件类都是使用此方式，并且在运行中被视作是优质资源。

Drupal全局类用于全局函数中。然而，Drupal 8的控制器、插件等是基于类的。其首先方式是将依赖的服务以构造函数参数的方式进行传递，或者以服务setter方法来注入，而不是调用全局的服务容器。

将某个对象依赖的服务显式传递，称为依赖注入。在很多情况下，依赖都是通过构造函数进行显式传递。如：[路由访问检测](https://www.drupal.org/node/2122195)在服务创建时插入当前用户，而当前请求对象则在访问检测时传递。你也可以通过setter方法来设置依赖。

**在全局函数中访问服务**

Drupal全局类提供静态方法来访问一些最常用的服务，如：Drupal::moduleHandler()返回模块处理程序服务，Drupal::translation()返回字符串翻译服务。如果你调用的服务没有专用方法，你也可以使用Drupal::service()来获取任何已定义的服务。

例子：使用专用方法\Drupal::database()来访问数据库服务

{% highlight php %}

    // Returns a Drupal\Core\Database\Connection object.
    $connection = \Drupal::database();
    $result = $connection->select('node', 'n')
      ->fields('n', array('nid'))
      ->execute();
      
{% endhighlight %}

例子：使用通用的\Drupal::service()方法来访问date服务

{% highlight php %}

    <?php
    // Returns a Drupal\Core\Datetime\Date object.
    $date = \Drupal::service('date');
    ?>
    
{% endhighlight %}

理想情况下，你应该尽量少的使用全局函数，并且重构控制器、事件监听、插件等。适当的方式应该是使用依赖注入；查看下面的文档。

在[Symfony 2的文档](http://symfony.com/doc/current/book/service_container.html)中均有代码范例。

**定义你的服务**

你可以使用example.services.yml文件来定义自己的服务，其中example就是你定义服务所在的模块名称。文件的格式与core.services.yml相同。

有多个子系统需要你定义服务，如：[自定义路由访问检测类](https://www.drupal.org/node/2122195)，[自定义参数转换](http://drupal.org/node/2122223)，或者[定义一个插件管理器](http://drupal.org/node/1637730)，这都需要将你的类注册为服务。

也可以使用 $GLOBALS['conf']['container_yamls']来增加用于定义服务的YAML文件，只是这样做十分少见。

**Drupal 7全局函数与Drupal 8服务对比**

我们用运行一个模块钩子所需要的代码来对比一下Drupal 7与8的区别。在Drupal中，你会使用module_invoke_all('help')来运行所有hook_help钩子，因为我们是在代码中直接调用module_invoke_all()函数，那么其他人就很难在不改变Drupal核心函数的情况下修改Drupal运行钩子的方式。

在Drupal 8中，用ModuleHandler代替了module_*函数。因此在Drupal 8中，你应该使用\Drupal::moduleHandler()->invokeAll('help')。在这个例子中\Drupal::moduleHandler()通过服务容器找出已注册的模块处理程序服务，然后调用服务的invokeAll()方法。

这种机制比Drupal 7的好，因为它允许Drupal 发行版、托管商或者其他模块通过修改注册的模块处理服务类，实现ModuleHandlerInterface接口，来覆写模块运行其他模块钩子的方式。这种修改对于其他部分的Drupal代码而言是透明的。这意味着Drupal的更多部分可以在不修改核心的情况下进行替换。代码的依赖也有更好的文档并且有关的边界也更好的分离。最好，服务可以通过接口进行更加简便的单元测试，而不是整合测试。

**对比Drupal 7全局变量与Drupal 8服务**

Drupal7的一些全局变量，如 global $language 与 global $user 在Drupal 8中也是通过服务进行访问（并且不再是全局变量）。请参考[Drupal::languageManager()->getLanguage(Language::TYPE_INTERFACE)](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21Language%21LanguageManager.php/function/LanguageManager%3A%3AgetLanguage/8) 与 [Drupal::currentUser()](https://api.drupal.org/api/drupal/core%21lib%21Drupal.php/function/Drupal%3A%3AcurrentUser/8)。


