---
title:  "Symfony服务容器 ( Symfony Service Container ) "
categories: PHP Symfony
---

这是一篇关于Symfony2服务容器实现的笔记，原文请查看文章末尾的相关资料部分。

上一篇笔记介绍了依赖注入容器，依赖注入容器使得对各种系统功能的调用变得简便，但同时引入了一个新的问题就是容器的编写与维护变得非常复杂。Symfony2试图通过ServiceContainer、ServiceContainerBuilder以及xml、yml配置文件来使容器的实现变得简单。

**ServiceContainer**

首先，引入sfServiceContainer类<http://svn.symfony-project.com/components/dependency_injection/trunk/lib/sfServiceContainer.php>，将上一篇笔记中的container作点修改，继承sfServiceContainer：

{% highlight php %}
    class Container extends sfServiceContainer
    {
      static protected $shared = array();
    
      protected function getMailTransportService()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => $this['mailer.username'],
          'password' => $this['mailer.password'],
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }
    
      protected function getMailerService()
      {
        if (isset(self::$shared['mailer']))
        {
          return self::$shared['mailer'];
        }
    
        $class = $this['mailer.class'];
    
        $mailer = new $class();
        $mailer->setDefaultTransport($this->getMailTransportService());
    
        return self::$shared['mailer'] = $mailer;
      }
    }
{% endhighlight %}

然后通过这种方式来调用：

{% highlight php %}
    require_once 'PATH/TO/sf/lib/sfServiceContainerAutoloader.php';
    sfServiceContainerAutoloader::register();
    
    $sc = new Container(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
      'mailer.class'    => 'Zend_Mail',
    ));
    
    $mailer = $sc->mailer;
{% endhighlight %}

通过实现__get方法，我们可以通过container对象属性的方式来调用各种服务，可以通过setService对容器进行配置，而需要我们编写的Container类变得更加格式化，都是由一系列get...Service方法组成。

**ServiceContainerBuilder**

因此我们可以通过一个ServiceContainerBuilder对ServiceContainer进行动态配置，sfServiceContainerBuilder: <http://svn.symfony-project.com/components/dependency_injection/trunk/lib/sfServiceContainerBuilder.php>

{% highlight php %}
    require_once 'PATH/TO/sf/lib/sfServiceContainerAutoloader.php';
    sfServiceContainerAutoloader::register();
    
    $sc = new sfServiceContainerBuilder();
    
    $sc->
      register('mail.transport', 'Zend_Mail_Transport_Smtp')->
      addArgument('smtp.gmail.com')->
      addArgument(array(
        'auth'     => 'login',
        'username' => '%mailer.username%',
        'password' => '%mailer.password%',
        'ssl'      => 'ssl',
        'port'     => 465,
      ))->
      setShared(false)
    ;
    
    $sc->
      register('mailer', '%mailer.class%')->
      addMethodCall('setDefaultTransport', array(new sfServiceReference('mail.transport')))

{% endhighlight %}
sfServiceContainerBuilder通过register函数来创建一个service，传入service的id以及对应的类，再通过addArgument传入将类实例化时传给构造函数的参数，通过addMethodCall来设置对象创建后调用的初始化函数，通过setShared来指定service是否全局共享。 sfServiceContainerBuilder继承自sfServiceContainer，覆写了getService方法，使得既可以调用静态设置的service，也可以使用动态添加的service。

**使用配置文件来描述Service**

既然service可以通过php动态添加，那么是否可以通过读取配置文件来动态生成呢？答案是肯定的。Symfony提供 sfServiceContainerLoader*** 类，通过读取配置文件，将配置文件转换成 sfServiceDefinition 并且对其进行配置，然后通过containerbuilder的setServiceDefinition方法来添加service，实现方式可以参考 <http://svn.symfony-project.com/components/dependency_injection/trunk/lib/sfServiceContainerLoader.php> 以及相关loader 的类。

**使用缓存提升性能**

使用配置文件来描述service非常强大而灵活，但会引入一个问题，如果每次运行都要读取配置文件，动态生成service，导致性能问题，而使用PHP代码扩展sfServiceContainer类则可以获得最大的性能优化。 Symfony除了有sfServiceContainerLoader来加载service以外，还可以通过sfServiceContainerDumper来将service的写入到文件中，譬如 sfServiceContainerDumperXml 可以生成上面的使用的配置文件，而sfServiceContainerDumperPhp则可以将service生成一个sfServiceContainer子类的代码。 通过这种方式，Symfony2可以将配置文件的灵活与PHP代码的性能很好的结合起来。

**Symfony2中的实现**

在Symfony2启动流程分析中，有以下的代码段：

{% highlight php %}
                $container = $this->buildContainer();
                $container->compile();
                $this->dumpContainer($cache, $container, $class, $this->getContainerBaseClass());
{% endhighlight %}
这就是Symfony对container的实现。

**相关资料：**

<http://svn.symfony-project.com/components/dependency_injection/trunk/lib/>

<http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container>

<http://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services>

<http://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services>

<http://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed>
