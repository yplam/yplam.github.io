---
title:  "Symfony 性能优化"
categories: PHP Symfony
---

测试环境 ubuntu 14.04, php 5.5.9，使用 ab -c 10 -t 5 的方式针对不同的Symfony配置进行性能测试。

首先，symfony new test 3.0 命令行创建3.0版本的symfony

**初始性能测试**

{% highlight php %}
Requests per second:    263.32 [#/sec] (mean)
Time per request:       37.977 [ms] (mean)
{% endhighlight %}

**使用APC**

{% highlight php %}
Requests per second:    307.96 [#/sec] (mean)
Time per request:       32.472 [ms] (mean)
{% endhighlight %}

{% highlight php %}
$loader = require __DIR__.'/../app/autoload.php';
include_once __DIR__.'/../var/bootstrap.php.cache';

$apcLoader = new Symfony\Component\ClassLoader\ApcClassLoader(sha1(__FILE__), $loader);
$loader->unregister();
$apcLoader->register(true);
{% endhighlight %}

在产线环境还可以对apc进行优化以提升性能，不过需要注意deploy后重启php-fpm
{% highlight php %}
apc.stat=0
{% endhighlight %}

**不使用twig，直接输出json**

{% highlight php %}
Requests per second:    372.09 [#/sec] (mean)
Time per request:       26.875 [ms] (mean)
{% endhighlight %}

```
use Symfony\Component\HttpFoundation\JsonResponse;

class DefaultController extends Controller
{
    /**
     * @Route("/", name="homepage")
     */
    public function indexAction(Request $request)
    {
        return new JsonResponse(array('hello'=>'world'));
    }
}

```


**禁用monolog**

如果你不需要在产线记录日志，或者你根本没有看过产线日志，那么禁用monolog也可以带来性能的提升。当然，使用redis log也是一个不错的选择。

{% highlight php %}
Requests per second:    405.40 [#/sec] (mean)
Time per request:       24.667 [ms] (mean)
{% endhighlight %}

**禁用SwiftMailer**

对于需要经常发送邮件的应用而言，SwiftMailer是一个不错的选择，然而SwiftMailer会监听kernel.terminate事件来发送邮件，这样就是对于没有发送邮件需求的请求，使用SwiftMailer还会产生4-5ms的时间消耗，对于性能要求高的应用，这也是不可接受的。

并且，就算是需要发送邮件，对于一个高性能的应用，将邮件放在队列中发送比在kernel.terminate中发送要更加合理一些。

{% highlight php %}
Requests per second:    506.43 [#/sec] (mean)
Time per request:       19.746 [ms] (mean)
{% endhighlight %}

以上就是暂时能够整理出来的Symfony性能最优化初始配置，当然，此配置只能做一些简单的restful应用。

在实际项目中，我们在创建新的service，添加新bundle时必须要关注其带来的性能影响，下面是一些常用的避免性能恶化的实践：

**避免在service构造函数中做过多操作**

特别是一些事件监听类服务，可能每次请求都会对该服务进行实例化操作，而实际上监听器只会在某些情况下才会进行真实的操作，因此那些消耗资源的初始化工作应该在真正用到前才进行。

**使用doctrine cache，适当选择使用dql或者orm**


