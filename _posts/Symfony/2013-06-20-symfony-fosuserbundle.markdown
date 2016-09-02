---
title:  "FOSUserBundle 入门"
categories: PHP Symfony
---

这是一篇介绍FOSUserBundle的安装与配置笔记，面向Symfony2.1+，更详细的内容请参考：<https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.md>

Symfony2的安全组件提供了一个具有扩展性的安全机制，允许你从配置文件、数据库或者任何你能想象到的地方加载用户。FOSUserBundle就是基于这种安全机制上建立的，为你提供一个简便的方式来将用户保存在数据库中。

**下载FOSUserBundle**

修改composer.json文件，在require部分增加以下内容：

{% highlight json %}

"friendsofsymfony/user-bundle": "~2.0@dev"

{% endhighlight %}

使用composer更新

{% highlight shell %}

$ php composer.phar update friendsofsymfony/user-bundle

{% endhighlight %}

**使能FOSUserBundle**

修改app/AppKernel.php文件，使能FOSUserBundle

{% highlight php %}
    <?php
    // app/AppKernel.php
    
    public function registerBundles()
    {
        $bundles = array(
            // ...
            new FOS\UserBundle\FOSUserBundle(),
        );
    }
{% endhighlight %}

**创建用户类**

创建一个自己的UserBundle，建立自己的User Entity。这样做有一个好处是以后可以用这个UserBundle来覆写FOSUserBundle。

{% highlight php %}
    <?php
    // src/Acme/UserBundle/Entity/User.php
    
    namespace Acme\UserBundle\Entity;
    
    use FOS\UserBundle\Model\User as BaseUser;
    use Doctrine\ORM\Mapping as ORM;
    
    /**
     * @ORM\Entity
     * @ORM\Table(name="fos_user")
     */
    class User extends BaseUser
    {
        /**
         * @ORM\Id
         * @ORM\Column(type="integer")
         * @ORM\GeneratedValue(strategy="AUTO")
         */
        protected $id;
    
        public function __construct()
        {
            parent::__construct();
            // your own logic
        }
    }

{% endhighlight %}
  
  
**配置security.yml**

{% highlight yml %}
    # app/config/security.yml
    security:
        encoders:
            FOS\UserBundle\Model\UserInterface: sha512
    
        role_hierarchy:
            ROLE_ADMIN:       ROLE_USER
            ROLE_SUPER_ADMIN: ROLE_ADMIN
    
        providers:
            fos_userbundle:
                id: fos_user.user_provider.username
    
        firewalls:
            main:
                pattern: ^/
                form_login:
                    provider: fos_userbundle
                    csrf_provider: form.csrf_provider
                logout:       true
                anonymous:    true
    
        access_control:
            - { path: ^/login$, role: IS_AUTHENTICATED_ANONYMOUSLY }
            - { path: ^/register, role: IS_AUTHENTICATED_ANONYMOUSLY }
            - { path: ^/resetting, role: IS_AUTHENTICATED_ANONYMOUSLY }
            - { path: ^/admin/, role: ROLE_ADMIN }

{% endhighlight %}

**配置FOSUserBundle**

{% highlight yml %}
    # app/config/config.yml
    fos_user:
        db_driver: orm # other valid values are 'mongodb', 'couchdb' and 'propel'
        firewall_name: main
        user_class: Your\UserBundle\Entity\User
{% endhighlight %}

user_class需要按照自己的需求来设置。

**加载路由配置文件**

{% highlight yml %}
    # app/config/routing.yml
    fos_user_security:
        resource: "@FOSUserBundle/Resources/config/routing/security.xml"
    
    fos_user_profile:
        resource: "@FOSUserBundle/Resources/config/routing/profile.xml"
        prefix: /profile
    
    fos_user_register:
        resource: "@FOSUserBundle/Resources/config/routing/registration.xml"
        prefix: /register
    
    fos_user_resetting:
        resource: "@FOSUserBundle/Resources/config/routing/resetting.xml"
        prefix: /resetting
    
    fos_user_change_password:
        resource: "@FOSUserBundle/Resources/config/routing/change_password.xml"
        prefix: /profile

{% endhighlight %}

需要注意的是，需要配置好SwiftmailerBundle后，重置密码等功能才能正常工作。

**更新数据库**

可以先看看是不是一切正常

{% highlight shell %}
    $ php app/console doctrine:schema:update --dump-sql
{% endhighlight %}

然后更新数据库

{% highlight shell %}
    $ php app/console doctrine:schema:update --force
{% endhighlight %}
如无意外，至此FOSUserBundle的功能就已经可以正常功能。可以尝试访问 /login /register 等进行测试。
