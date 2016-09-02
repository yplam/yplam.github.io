---
title:  "依赖注入容器（Dependency Injection Container）"
categories: PHP Symfony
---

这是一篇关于Symfony依赖注入容器 Dependency Injection Container 的笔记，原文地址：<http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container>

上一篇文章介绍过依赖注入，通过一种非常简单的方式很好的降低了类之间的耦合。但是当业务中使用的类原来越多，譬如在很多地方都需要用到User，每次创建User时都需要记住传递给User的每个参数，这是不可能的，因此就需要用到依赖注入容器。 A Dependency Injection Container is an object that knows how to instantiate and configure objects. And to be able to do its job, it needs to knows about the constructor arguments and the relationships between the objects. 依赖注入容器的作用就是初始化与配置各类对象，管理对象间的依赖关系。下面是一个简单的实现：

{% highlight php %}
    class Container
    {
      public function getMailTransport()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => 'foo',
          'password' => 'bar',
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }
    
      public function getMailer()
      {
        $mailer = new Zend_Mail();
        $mailer->setDefaultTransport($this->getMailTransport());
    
        return $mailer;
      }
    }
{% endhighlight %}

使用起来也很简便：

{% highlight php %}
    $container = new Container();
    $mailer = $container->getMailer();
{% endhighlight %}

这样，系统的所有功能都是通过container来统一管理。为了是container更加实用，需要添加一些配置参数：

{% highlight php %}
    class Container
    {
      protected $parameters = array();
    
      public function __construct(array $parameters = array())
      {
        $this->parameters = $parameters;
      }
    
      public function getMailTransport()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => $this->parameters['mailer.username'],
          'password' => $this->parameters['mailer.password'],
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }
    
      public function getMailer()
      {
        $mailer = new Zend_Mail();
        $mailer->setDefaultTransport($this->getMailTransport());
    
        return $mailer;
      }
    }
{% endhighlight %}

然后这样初始化：

{% highlight php %}
    $container = new Container(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
    ));
    $mailer = $container->getMailer();
{% endhighlight %}

为了不用每次使用某个功能时都新建一个对象，再对container进行一些改进：

{% highlight php %}
    class Container
    {
      static protected $shared = array();
    
      // ...
    
      public function getMailer()
      {
        if (isset(self::$shared['mailer']))
        {
          return self::$shared['mailer'];
        }
    
        $class = $this->parameters['mailer.class'];
    
        $mailer = new $class();
        $mailer->setDefaultTransport($this->getMailTransport());
    
        return self::$shared['mailer'] = $mailer;
      }
    }
{% endhighlight %}
使用依赖注入容器后，对系统功能的初始化与调用变得非常简单，但同时很明显的，当系统变得庞大后，创建与维护Contianer类将是一个噩梦，因此Symfony2提供了一个ContainerBuilder类，根据用户的配置自动生成Container类的代码。

**参考资料：** 

<http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container>
