---
title:  "SonataAdminBundle创建自定义后台页面"
categories: PHP Symfony
---

SonataAdminBundle可以非常简便地实现CRUD页面，然而，如果我们需要在后台中实现与Entity无关的管理页面，应该怎么办呢？下面分享自己在摸索工程中的一个实现方法。

首先 Google 一番，发现已经有人遇到过这个问题，并且提供了一些解决方法：

[Custom page & controller in Sonata Admin Bundle](http://blog.eike.se/2014/03/custom-page-controller-in-sonata-admin.html)

[The page which had not been connect with entity, in SonataAdminBundle](http://sysmagazine.com/posts/167543/)

[sonataadminbundle创建一个自定义页面](http://www.newlifeclan.com/symfony/archives/547)

然而，上面的方法均有一个特点，就是通过覆写SonataAdminBundle的dashboard模板来将自定义页面的链接加进去。身为一名具有处女座精神的程序猿，这种方法当然不能接受 :)

查看代码，发现SonataAdminBundle的实现方式也很简单：**实现一个Block，然后将Block配置到SonataAdmin中。**

**创建Block**

假设我们需要创建一个Block来显示管理页面的链接，那么我们可以添加一个区块服务：

services.yml

```
    app.admin.block.manager_list:
        class: AppBundle\Block\AdminManagerBlockService
        tags:
            - { name: sonata.block }
        arguments:
            - app.admin.block.manager_list
            - @templating

```

**在config.yml中配置这个区块：**

```
sonata_block:
    default_contexts: [cms]
    blocks:
        # Enable the SonataAdminBundle block
        sonata.admin.block.admin_list:
            contexts:   [admin]
        app.admin.block.manager_list:
            contexts:   [admin]

sonata_admin:
    title: Admin
    dashboard:
        blocks:
            -
                position: left
                type: sonata.admin.block.admin_list
            -
                position: top
                type: app.admin.block.manager_list

```

**实现Block内容输出**

```
namespace AppBundle\Block;

use Sonata\BlockBundle\Block\BlockContextInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Bundle\FrameworkBundle\Templating\EngineInterface;

use Sonata\AdminBundle\Form\FormMapper;
use Sonata\AdminBundle\Validator\ErrorElement;
use Sonata\AdminBundle\Admin\Pool;

use Sonata\BlockBundle\Model\BlockInterface;
use Sonata\BlockBundle\Block\BaseBlockService;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\OptionsResolver\OptionsResolverInterface;

/**
 *
 */
class AdminManagerBlockService extends BaseBlockService
{


    /**
     * @param string                                                     $name
     * @param \Symfony\Bundle\FrameworkBundle\Templating\EngineInterface $templating
     */
    public function __construct($name, EngineInterface $templating)
    {
        parent::__construct($name, $templating);
    }

    /**
     * {@inheritdoc}
     */
    public function execute(BlockContextInterface $blockContext, Response $response = null)
    {

        $settings = $blockContext->getSettings();


        return $this->renderPrivateResponse('AppBundle:Block:manager_list.html.twig', array(
            'block'         => $blockContext->getBlock(),
            'settings'      => $settings,
        ), $response);
    }

    /**
     * {@inheritdoc}
     */
    public function validateBlock(ErrorElement $errorElement, BlockInterface $block)
    {
        // TODO: Implement validateBlock() method.
    }

    /**
     * {@inheritdoc}
     */
    public function buildEditForm(FormMapper $formMapper, BlockInterface $block)
    {

    }

    /**
     * {@inheritdoc}
     */
    public function getName()
    {
        return 'Admin Manager';
    }

    /**
     * {@inheritdoc}
     */
    public function setDefaultSettings(OptionsResolverInterface $resolver)
    {
        $resolver->setDefaults(array(
            'groups' => false
        ));

        $resolver->setAllowedTypes(array(
            'groups' => array('bool', 'array')
        ));
    }
}

```

**继承SonataAdminBundle的模板**

```
{% raw %}
{% extends sonata_block.templates.block_base %}

{% block block %}
    <div class="box">
        <div class="box-header">
            <h3 class="box-title">Manager</h3>
        </div>
        <div class="box-body">
            <table class="table table-hover">
                <tbody>
                    <tr>
                        <td class="sonata-ba-list-label" width="40%">
                            统计数据
                        </td>
                        <td>
                            <div class="btn-group">
                                <a class="btn btn-link btn-flat" href="">
                                    <i class="glyphicon glyphicon-list"></i>List
                                </a>
                            </div>
                        </td>
                    </tr>
                    
                </tbody>
            </table>
        </div>
    </div>
{% endblock %}
{% endraw %}
```

这样，就可以在SonataAdminBundle的Dashboard页面头部添加自己的管理区块。
