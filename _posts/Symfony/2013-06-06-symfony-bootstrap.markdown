---
title:  "Symfony2 启动流程分析"
categories: PHP Symfony
---

**入口文件app.php**

{% highlight php %}
    <?php
    
    use Symfony\Component\ClassLoader\ApcClassLoader;
    use Symfony\Component\HttpFoundation\Request;
    
    $loader = require_once __DIR__.'/../app/bootstrap.php.cache';
    
    // Use APC for autoloading to improve performance.
    // Change 'sf2' to a unique prefix in order to prevent cache key conflicts
    // with other applications also using APC.
    /*
    $loader = new ApcClassLoader('sf2', $loader);
    $loader->register(true);
    */
    
    require_once __DIR__.'/../app/AppKernel.php';
    //require_once __DIR__.'/../app/AppCache.php';
    
    $kernel = new AppKernel('prod', false);
    $kernel->loadClassCache();
    //$kernel = new AppCache($kernel);
    Request::enableHttpMethodParameterOverride();
    $request = Request::createFromGlobals();
    $response = $kernel->handle($request);
    $response->send();
    $kernel->terminate($request, $response);
{% endhighlight %}
入口文件app_dev.php

{% highlight php %}
    <?php
    
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\Debug\Debug;
    
    // If you don't want to setup permissions the proper way, just uncomment the following PHP line
    // read http://symfony.com/doc/current/book/installation.html#configuration-and-setup for more information
    //umask(0000);
    
    // This check prevents access to debug front controllers that are deployed by accident to production servers.
    // Feel free to remove this, extend it, or make something more sophisticated.
    if (isset($_SERVER['HTTP_CLIENT_IP'])
        || isset($_SERVER['HTTP_X_FORWARDED_FOR'])
        || !in_array(@$_SERVER['REMOTE_ADDR'], array('127.0.0.1', 'fe80::1', '::1'))
    ) {
        header('HTTP/1.0 403 Forbidden');
        exit('You are not allowed to access this file. Check '.basename(__FILE__).' for more information.');
    }
    
    $loader = require_once __DIR__.'/../app/bootstrap.php.cache';
    Debug::enable();
    
    require_once __DIR__.'/../app/AppKernel.php';
    
    $kernel = new AppKernel('dev', true);
    $kernel->loadClassCache();
    Request::enableHttpMethodParameterOverride();
    $request = Request::createFromGlobals();
    $response = $kernel->handle($request);
    $response->send();
    $kernel->terminate($request, $response);
{% endhighlight %}
app_dev.php相对于app.php的主要区别在于 environment 与 debug 参数。

**app/bootstrap.php.cache：** 

bootstrap.php.cache 在运行composer update或者composer install 的时候由脚本生成，主要的功能是将Symfony中必须要用到的核心类打包到单个文件中，同时提供classloader。

**app/AppKernel.php**

扩展自Kernel类，对系统进行配置。创建AppKernel的时候会传入environment 与 debug 参数，environment参数会影响配置文件路径，缓存文件路径，debug参数会影响到缓存文件路径以及系统对cache的处理。

{% highlight php %}
    <?php
    
    use Symfony\Component\HttpKernel\Kernel;
    use Symfony\Component\Config\Loader\LoaderInterface;
    
    class AppKernel extends Kernel
    {
        public function registerBundles()
        {
            $bundles = array(
                new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
                new Symfony\Bundle\SecurityBundle\SecurityBundle(),
                new Symfony\Bundle\TwigBundle\TwigBundle(),
                new Symfony\Bundle\MonologBundle\MonologBundle(),
                new Symfony\Bundle\SwiftmailerBundle\SwiftmailerBundle(),
                new Symfony\Bundle\AsseticBundle\AsseticBundle(),
                new Doctrine\Bundle\DoctrineBundle\DoctrineBundle(),
                new Sensio\Bundle\FrameworkExtraBundle\SensioFrameworkExtraBundle(),
            );
    
            if (in_array($this->getEnvironment(), array('dev', 'test'))) {
                $bundles[] = new Acme\DemoBundle\AcmeDemoBundle();
                $bundles[] = new Symfony\Bundle\WebProfilerBundle\WebProfilerBundle();
                $bundles[] = new Sensio\Bundle\DistributionBundle\SensioDistributionBundle();
                $bundles[] = new Sensio\Bundle\GeneratorBundle\SensioGeneratorBundle();
            }
    
            return $bundles;
        }
    
        public function registerContainerConfiguration(LoaderInterface $loader)
        {
            $loader->load(__DIR__.'/config/config_'.$this->getEnvironment().'.yml');
        }
    }
{% endhighlight %}
**$kernel->loadClassCache();** 

做一个需要加载cache的标记，但不会进行加载工作。

**Request::enableHttpMethodParameterOverride()**

是否允许通过发送的参数来修改请求的方法，只对POST有效。

**$request = Request::createFromGlobals();**

创建request对象，通过获取$_GET,$_POST等数据，创建request对象。

**$response = $kernel->handle($request);** 

处理请求。symfony内核在此处启动。

{% highlight php %}
        public function handle(Request $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
        {
            if (false === $this->booted) {
                $this->boot();
            }
    
            return $this->getHttpKernel()->handle($request, $type, $catch);
        }
{% endhighlight %}
boot方法启动内核，进行初始化配置

{% highlight php %}
        public function boot()
        {
            if (true === $this->booted) {
                return;
            }
    
            if ($this->loadClassCache) {
                $this->doLoadClassCache($this->loadClassCache[0], $this->loadClassCache[1]);
            }
    
            // init bundles
            $this->initializeBundles();
    
            // init container
            $this->initializeContainer();
    
            foreach ($this->getBundles() as $bundle) {
                $bundle->setContainer($this->container);
                $bundle->boot();
            }
    
            $this->booted = true;
        }
{% endhighlight %}
doLoadClassCache会根据前面loadClassCache的调用，将加载cache目录下的classes.php，如果缓存不存在，则根据classes.map文件来生成。如果debug=true则还会检测每个classes对应原文件的修改时间来判断是否需要重新生成。 初始化bundle，主要是处理bundle的继承关系

{% highlight php %}
        /**
         * Initializes the data structures related to the bundle management.
         *
         *  - the bundles property maps a bundle name to the bundle instance,
         *  - the bundleMap property maps a bundle name to the bundle inheritance hierarchy (most derived bundle first).
         *
         * @throws \LogicException if two bundles share a common name
         * @throws \LogicException if a bundle tries to extend a non-registered bundle
         * @throws \LogicException if a bundle tries to extend itself
         * @throws \LogicException if two bundles extend the same ancestor
         */
        protected function initializeBundles()
        {
            // init bundles
            $this->bundles = array();
            $topMostBundles = array();
            $directChildren = array();
    
            foreach ($this->registerBundles() as $bundle) {
                $name = $bundle->getName();
                if (isset($this->bundles[$name])) {
                    throw new \LogicException(sprintf('Trying to register two bundles with the same name "%s"', $name));
                }
                $this->bundles[$name] = $bundle;
    
                if ($parentName = $bundle->getParent()) {
                    if (isset($directChildren[$parentName])) {
                        throw new \LogicException(sprintf('Bundle "%s" is directly extended by two bundles "%s" and "%s".', $parentName, $name, $directChildren[$parentName]));
                    }
                    if ($parentName == $name) {
                        throw new \LogicException(sprintf('Bundle "%s" can not extend itself.', $name));
                    }
                    $directChildren[$parentName] = $name;
                } else {
                    $topMostBundles[$name] = $bundle;
                }
            }
    
            // look for orphans
            if (count($diff = array_values(array_diff(array_keys($directChildren), array_keys($this->bundles))))) {
                throw new \LogicException(sprintf('Bundle "%s" extends bundle "%s", which is not registered.', $directChildren[$diff[0]], $diff[0]));
            }
    
            // inheritance
            $this->bundleMap = array();
            foreach ($topMostBundles as $name => $bundle) {
                $bundleMap = array($bundle);
                $hierarchy = array($name);
    
                while (isset($directChildren[$name])) {
                    $name = $directChildren[$name];
                    array_unshift($bundleMap, $this->bundles[$name]);
                    $hierarchy[] = $name;
                }
    
                foreach ($hierarchy as $bundle) {
                    $this->bundleMap[$bundle] = $bundleMap;
                    array_pop($bundleMap);
                }
            }
    
        }
{% endhighlight %}
初始化container，为symfony的核心，原理与实现方式请参考下面的文章。基本流程为：如果container缓存存在，则直接使用缓存，如果不存在则创建container，获取每个bundle与container相关的信息，加载配置文件，配置每个bundle，打包生成缓存。 [Introduction to the Symfony Service Container](http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container) [Dependency Injection](http://symfony.com/doc/current/components/dependency_injection/index.html)  

{% highlight php %}
        /**
         * Initializes the service container.
         *
         * The cached version of the service container is used when fresh, otherwise the
         * container is built.
         */
        protected function initializeContainer()
        {
            $class = $this->getContainerClass();
            $cache = new ConfigCache($this->getCacheDir().'/'.$class.'.php', $this->debug);
            $fresh = true;
            if (!$cache->isFresh()) {
                $container = $this->buildContainer();
                $container->compile();
                $this->dumpContainer($cache, $container, $class, $this->getContainerBaseClass());
    
                $fresh = false;
            }
    
            require_once $cache;
    
            $this->container = new $class();
            $this->container->set('kernel', $this);
    
            if (!$fresh && $this->container->has('cache_warmer')) {
                $this->container->get('cache_warmer')->warmUp($this->container->getParameter('kernel.cache_dir'));
            }
        }
{% endhighlight %}
getHttpKernel()->handle()，此时可以通过contianer来获取http_kernel服务对请求进行处理。 $response->send(); 发送内容 $kernel->terminate($request, $response); 结束  
