---
title:  "依赖注入 ( Dependency Injection )"
categories: PHP Symfony
---

这是一篇关于Symfony依赖注入（Dependency Injection）的笔记，原文地址：<http://fabien.potencier.org/article/11/what-is-dependency-injection>

假设需要实现下面的需求：**编写一个User类，需要实现Session的相关操作** 下面是一种简单的实现方式：

{% highlight php %}
    class SessionStorage
    {
      function __construct($cookieName = 'PHP_SESS_ID')
      {
        session_name($cookieName);
        session_start();
      }
    
      function set($key, $value)
      {
        $_SESSION[$key] = $value;
      }
    
      function get($key)
      {
        return $_SESSION[$key];
      }
    
      // ...
    }
    
    class User
    {
      protected $storage;
    
      function __construct()
      {
        $this->storage = new SessionStorage();
      }
    
      function setLanguage($language)
      {
        $this->storage->set('language', $language);
      }
    
      function getLanguage()
      {
        return $this->storage->get('language');
      }
    
      // ...
    }
{% endhighlight %}

我们可以这样调用上面的代码：

{% highlight php %}
    $user = new User();
    $user->setLanguage('fr');
    $user_language = $user->getLanguage();
{% endhighlight %}

但这里有一个问题：代码中将User的storage属性写死了，第三方调用User时无法对storage进行配置。因此，对User代码进行下面的修改：

{% highlight php %}
    class User
    {
      function __construct($storage)
      {
        $this->storage = $storage;
      }
    
      // ...
    }
{% endhighlight %}

不在User类中创建SessionStorage对象，而是将storage对象作为参数传递给User的构造函数，这就是依赖注入。 "Dependency Injection is where components are given their dependencies through their constructors, methods, or directly into fields." 上面就是依赖注入的定义，根据其定义，依赖注入可以通过下面三种方式进行：

**构造器注入 Constructor Injection：**

{% highlight php %}
    class User
    {
      function __construct($storage)
      {
        $this->storage = $storage;
      }
    
      // ...
    }
{% endhighlight %}

**设值注入 Setter Injection：**

{% highlight php %}
    class User
    {
      function setSessionStorage($storage)
      {
        $this->storage = $storage;
      }
    
      // ...
    }
{% endhighlight %}

**属性 Property Injection：**

{% highlight php %}
    class User
    {
      public $sessionStorage;
    }
    
    $user->sessionStorage = $storage;
{% endhighlight %}

**相关资料：** 

<http://fabien.potencier.org/article/11/what-is-dependency-injection> 

<http://www.martinfowler.com/articles/injection.html> 

<http://www.cnblogs.com/xingyukun/archive/2007/10/20/931331.html>
