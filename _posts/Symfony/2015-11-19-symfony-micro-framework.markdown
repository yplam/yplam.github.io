---
title:  "Symfony2 太“重”了？试试 Symfony2 微框架"
categories: PHP Symfony
---

Symfony2 是一个全功能的框架，因此有些开发人员会以它太“重”了（而不是太“难”了）为理由而拒绝使用。在开发环境下，一个简单的Symfony页面大概需要花费 70ms，而prod环境下大概需要30ms（以上是个人在一个Web App上实践后数据）。虽然相对其提供的功能而言，这是一个可观的数据，但对于一个简单的REST API请求而言，这也许真的太“重”了。

Symfony 2.8 引入了微内核的功能，用来创建简单，甚至是单文件的Symfony 应用。

下面是一个简单的例子：

```

// app/MicroKernel.php
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\Config\Loader\LoaderInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Kernel;
use Symfony\Component\Routing\RouteCollectionBuilder;

class MicroKernel extends Kernel
{
    use MicroKernelTrait;

    public function registerBundles()
    {
        return array(new Symfony\Bundle\FrameworkBundle\FrameworkBundle());
    }

    protected function configureRoutes(RouteCollectionBuilder $routes)
    {
        $routes->add('/', 'kernel:indexAction', 'index');
    }

    protected function configureContainer(ContainerBuilder $c, LoaderInterface $loader)
    {
        $c->loadFromExtension('framework', ['secret' => '12345']);
    }

    public function indexAction()
    {
        return new Response('Hello World');
    }
}

```

MicroKernel类主要实现配置 bundle、配置路由、路由响应的功能；只需要简单几行代码就完成一个Symfony应用。当然，MicroKernel实际上不会提升Symfony的运行性能，它只不过是改变了Symfony配置路由与bundle的方式，但是这样的改变使得你的应用只使用Symfony最基本的功能，禁用了大量的特性，因此可以带来性能的提升：

![全框架与微框架]({{ site.url }}/assets/2015/symfony_micro.jpg "全框架与微框架")

显然，这样做带来一个非常大的灵活性，你可以在Symfony的基础上开发你的应用，避免通常微框架所带来的限制，当你的应用越来越复杂的时候，你可以逐渐的将需要用到的Symfony特性加进去。

譬如，下面的例子就是在上面的基础上添加了Twig模板、Web调试工具栏的支持，依然可以写在一个文件里。


```

// app/MicroKernel.php

class MicroKernel extends Kernel
{
    use MicroKernelTrait;

    public function registerBundles()
    {
        $bundles = array(
            new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
            new Symfony\Bundle\TwigBundle\TwigBundle(),
        );

        if (in_array($this->getEnvironment(), array('dev', 'test'), true)) {
            $bundles[] = new Symfony\Bundle\WebProfilerBundle\WebProfilerBundle();
        }

        return $bundles;
    }

    protected function configureRoutes(RouteCollectionBuilder $routes)
    {
        $routes->mount('/_wdt', $routes->import('@WebProfilerBundle/Resources/config/routing/wdt.xml'));
        $routes->mount('/_profiler', $routes->import('@WebProfilerBundle/Resources/config/routing/profiler.xml'));

        $routes->add('/', 'kernel:indexAction', 'index');
    }

    protected function configureContainer(ContainerBuilder $c, LoaderInterface $loader)
    {
        // load bundles' configuration
        $c->loadFromExtension('framework', [
            'secret' => '12345',
            'profiler' => null,
            'templating' => ['engines' => ['twig']],
        ]);

        $c->loadFromExtension('web_profiler', ['toolbar' => true]);

        // add configuration parameters
        $c->setParameter('mail_sender', 'user@example.com');

        // register services
        $c->register('app.markdown', 'AppBundle\\Service\\Parser\\Markdown');
    }

    public function indexAction()
    {
        return $this->container->get('templating')->renderResponse('index.html.twig');
    }
}

```

单文件应用可能会导致代码显得混乱，下面是一个可以在真实场景中使用的例子，包括单个service.yml、config.yml配置，路由使用 annotations 注释：

```

// app/MicroKernel.php

// ...

class MicroKernel extends Kernel
{
    use MicroKernelTrait;

    public function registerBundles()
    {
        return array(
            new Sensio\Bundle\FrameworkExtraBundle\SensioFrameworkExtraBundle(),
            new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
            new Symfony\Bundle\TwigBundle\TwigBundle(),
            new AppBundle\AppBundle(),
        );
    }

    protected function configureRoutes(RouteCollectionBuilder $routes)
    {
        $routes->mount('/', $routes->import('@AppBundle/Controller', 'annotation'));
    }

    protected function configureContainer(ContainerBuilder $c, LoaderInterface $loader)
    {
        $loader->load(__DIR__.'/config/config_'.$this->getEnvironment().'.yml');
        $loader->load(__DIR__.'/config/services.yml');
    }
}

```

当然，不要忘记在入口文件中使用MicroKernel :)

```
// web/app.php
use Symfony\Component\HttpFoundation\Request;

$loader = require __DIR__.'/../app/autoload.php';
require_once __DIR__.'/../app/MicroKernel.php';

$app = new MicroKernel('prod', false);
$app->loadClassCache();

$app->handle(Request::createFromGlobals())->send();

```

Symfony2引入微内核，还会带来另外一个便利，通常，对于后台、内容编辑、内容提交的页面，我们需要全功能的支持，然而对于一些只需要简单的JSON API接口，我们可以添加多一个使用微内核的入口文件，提高接口的响应速度，减少资源消耗。


