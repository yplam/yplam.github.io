---
title:  "使用twigExtension实现页面区块功能"
categories: PHP Symfony
---

所谓区块，就是在页面中展示的一块内容，可以与当前页面内容相关或者无关。譬如常见的在网站侧边栏展示
“最新文章”区块，或者在文章内容页展示“相关文章”区块。

对于与当前页面相关的区块，我们可以在页面的控制器中通过查询数据库等方式获取数据，在模板中
渲染；而对于与当前页面内容无关的区块，我们则需要通过其他方式。

Symfony在处理这些区块时，提供两种种方式： 

(来源：http://symfony.com/doc/current/book/templating.html)

**include，用来渲染页面相关内容**

```
{% raw  %}
{# app/Resources/views/article/list.html.twig #}
{% extends 'layout.html.twig' %}

{% block body %}
    <h1>Recent Articles<h1>

    {% for article in articles %}
        {{ include('article/article_details.html.twig', { 'article': article }) }}
    {% endfor %}
{% endblock %}
{% endraw %}
```

**render，用来渲染页面无关内容**

```
{% raw  %}

{# app/Resources/views/base.html.twig #}

{# ... #}
<div id="sidebar">
    {{ render(controller(
        'AcmeArticleBundle:Article:recentArticles',
        { 'max': 3 }
    )) }}
</div>

{% endraw %}
```


然而，由render的运行原理可以看出，它会发送一个子请求，调用子控制器。虽然官方说它很快，但是测试过后发现有很明显的时间消耗。

**使用Twig扩展函数**

Twig扩展函数是另一个很好的输出区块内容的方式（虽然Symfony官方的介绍不多）。

譬如：


```
<?php

namespace AppBundle\Twig\Extension;

use Symfony\Component\HttpKernel\KernelInterface;
use JMS\DiExtraBundle\Annotation as DI;

/**
 * @DI\Service("app.twig.extension", public=false)
 * @DI\Tag("twig.extension")
 */
class AppExtension extends \Twig_Extension {

    /**
     * {@inheritdoc}
     */
    public function getFunctions() {
        return array(
            'app_sidemenu' => new \Twig_Function_Method($this, 'renderSideMenu', array(
                'is_safe' => array('html'),
                'needs_environment' => true,
            )),
        );
    }


    /**
     * 侧边栏信息
     */
    public function renderSideMenu (  \Twig_Environment $twig ) {

        $categories = array();
        
        /* 在此处处理区块逻辑 */
        
        $template = $twig->loadTemplate('AppBundle:Common:sideblock.html.twig');
        $content = $template->render(array('categories' => $categories));
        return $content;
    }



    /**
     * {@inheritdoc}
     */
    public function getName() {
        return 'app_bundle';
    }
}


```

上面是一个简单的Twig扩展，其作用就是提供app_sidemenu函数，在页面中显示网站侧边栏分类菜单。
renderSideMenu方法从数据库中获取分类信息，用twig模板渲染成菜单html，然后缓存到redis里。

需要注意的是 getFunctions 方法中，返回数据的 needs_environment 参数，如果设为true，它会为你的回调函数
传入 \Twig_Environment $twig，使你在回调函数中方便地进行模板渲染。

使用 twigExtension 来渲染区块还有个好处就是可以很方便的管理缓存，因为我们可以将render出来的内容存在cache中，这样不同的区块可以有自己的缓存清理逻辑。

下面我们尝试在扩展中使用Redis来缓存区块内容。

```

namespace AppBundle\Twig\Extension;

use Symfony\Component\HttpKernel\KernelInterface;
use JMS\DiExtraBundle\Annotation as DI;

/**
 * @DI\Service("app.twig.site.extension")
 * @DI\Tag("twig.extension")
 */
class AppSiteExtension extends \Twig_Extension {

    protected $redis;
    
    /**
     * @DI\InjectParams({
     *     "redis" = @DI\Inject("snc_redis.cache")
     * })
     */
    public function __construct($redis){
        $this->redis = $redis;
    }

    /**
     * {@inheritdoc}
     */
    public function getFunctions() {
        return array(
            'app_header' => new \Twig_Function_Method($this, 'renderHeader', array(
                'is_safe' => array('html'),
                'needs_environment' => true,
            )),
        );
    }

    /**
     * 缓存头部信息，缓存时间 36000s
     */
    public function renderHeader( \Twig_Environment $twig){
        $key = 'app:cache:header';

        if($content = $this->redis->get($key)){
            return $content;
        }
        $template = $twig->loadTemplate('AppBundle:Common:header.html.twig');
        $content = $template->render(array());
        $this->redis->setex($key, 36000, $content);
        return $content;
    }

    /**
     * {@inheritdoc}
     */
    public function getName() {
        return 'app_bundle_site';
    }
}

```

上面是一个使用redis缓存区块内容的简单应用，跟yii的widget缓存类似（对yii了解不多，错了请指正）。

我们在扩展的构造函数中注入 $redis 实例，然后将渲染后的网站头部模板内容缓存到rendis中，在下一次访问中直接输出。

ps:在此只是给缓存信息一个定期的过期时间，实际应用场景中对cache的处理可能会更复杂。
