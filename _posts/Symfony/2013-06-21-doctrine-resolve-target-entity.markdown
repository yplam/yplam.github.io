---
title:  "如何通过抽象类与接口来定义关联"
categories: PHP Symfony
---

Symfony2 文档翻译：<http://symfony.com/doc/current/cookbook/doctrine/resolve_target_entity.html>

使用代码包（Bundle）的其中一个目标就是使相互间的依赖更少，这样用户在在其他应用中使用这个功能，而不许要包含不必要的代码。 Doctrine 2.2 中包含了一个新的功能： ``ResolveTargetEntityListener``, 通过此功能你可以拦截Doctrine中的某些调用，在运行时覆写Doctrine 实体（Entity）映射配置中的 ``targetEntity`` 参数。这样，在你的代码包中可以使用接口或者抽象类来做实体映射然后在运行时指定正确的 Entity。 这个功能的作用在于你可以通过配置的方式来实现实体见的关系，而不需要在代码包中写死。  

**背景**

假设你有一个`InvoiceBundle`，提供与发票相关的功能；一个`CustomerBundle`，提供用户管理的功能。你希望将这两个代码包分开，这样在其他系统中他们可以独立使用，但是当前的应用需要同时用到这两个功能。 在这种情况下，你可以定义一个``Invoice``实体，与之关联的是一个不存在的实体：``InvoiceSubjectInterface``。这样做的目的是可以用一个实现了以上接口的实体``ResolveTargetEntityListener`` 来代替掉``InvoiceSubjectInterface``。  

**实现**

下面用一个简单的例子来示范如何实现ResolveTargetEntityListener。 Customer 实体::

{% highlight php %}

        // src/Acme/AppBundle/Entity/Customer.php
    
        namespace Acme\AppBundle\Entity;
    
        use Doctrine\ORM\Mapping as ORM;
        use Acme\CustomerBundle\Entity\Customer as BaseCustomer;
        use Acme\InvoiceBundle\Model\InvoiceSubjectInterface;
    
        /**
         * @ORM\Entity
         * @ORM\Table(name="customer")
         */
        class Customer extends BaseCustomer implements InvoiceSubjectInterface
        {
            // In our example, any methods defined in the InvoiceSubjectInterface
            // are already implemented in the BaseCustomer
            // 假设 BaseCustomer 已经实现了InvoiceSubjectInterface
        }
{% endhighlight %}

Invoice 实体::

{% highlight php %}
        // src/Acme/InvoiceBundle/Entity/Invoice.php
    
        namespace Acme\InvoiceBundle\Entity;
    
        use Doctrine\ORM\Mapping AS ORM;
        use Acme\InvoiceBundle\Model\InvoiceSubjectInterface;
    
        /**
         * Represents an Invoice.
         *
         * @ORM\Entity
         * @ORM\Table(name="invoice")
         */
        class Invoice
        {
            /**
             * @ORM\ManyToOne(targetEntity="Acme\InvoiceBundle\Model\InvoiceSubjectInterface")
             * @var InvoiceSubjectInterface
             */
            protected $subject;
        }
        
{% endhighlight %}

InvoiceSubjectInterface 接口::

{% highlight php %}
        // src/Acme/InvoiceBundle/Model/InvoiceSubjectInterface.php
    
        namespace Acme\InvoiceBundle\Model;
    
        /**
         * An interface that the invoice Subject object should implement.
         * In most circumstances, only a single object should implement
         * this interface as the ResolveTargetEntityListener can only
         * change the target to a single object.
         * InvoiceSubject需要实现的接口
         */
        interface InvoiceSubjectInterface
        {
            // 接口需要实现的方法
    
            /**
             * @return string
             */
            public function getName();
        }
{% endhighlight %}

然后，你需要配置监听器（Listener），配置好替换关系

{% highlight yml %}
        # app/config/config.yml
            doctrine:
                # ....
                orm:
                    # ....
                    resolve_target_entities:
                        Acme\InvoiceBundle\Model\InvoiceSubjectInterface: Acme\AppBundle\Entity\Customer
{% endhighlight %}

 **总结**

有了 ``ResolveTargetEntityListener``，代码包间可以更好的解藕。使代码包的功能更加独立，同时又可以通过这种方式来定义不同实体间的关系，使代码包更加容易维护。
