---
title:  "使用Capifony对Symfony网站进行发布"
categories: PHP Symfony
---

以前一直使用Drupal来搭建网站，由于需要定制的代码量不是很大，所以每次修改都是直接用sftp传到服务器，再到Drupal后台清一下缓存。最近使用Symfony2进行开发，到了发布环节遇到了问题。Symfony代码的发布并不像其他CMS那么方便，因为更新代码后还需要assetic:dump，assets:install，cache:clear等一系列操作，因此每次更新往往需要登录到服务器去更新代码，运行清理cache的一系列命令。 [capifony](http://capifony.org/)是针对Symfony开发的应用部署脚本，基于[Capistrano](https://github.com/capistrano/capistrano)。使用capifony只需要进行简单的配置，就可以很方便的将代码部署到不同的服务器上。 


**安装**

{% highlight shell %}
gem install capifony
{% endhighlight %}

**设置项目**

在项目中生成初始化配置

{% highlight shell %}
cd path/to/your/project
capifony .
{% endhighlight %}

这会在你的项目根目录中生成一个Capfile文件，在config/（Symfony 1.x）或者app/config/(Symfony 2)目录中生成deploy.rb配置文件。

**配置**

首先，你需要选择你的发布方式，然后将你的服务器的连接信息填到config/deploy.rb (或者 app/config/deploy.rb)。

a) deployment → scm → production

第一种方式是通过git库来发布到产线，在这种情况下，产线服务器必须可以访问到git库并且可以运行pull；你还必须可以在发布服务器上通过ssh访问产线服务器。

{% highlight ruby %}
# deploy.rb

set   :application,   "My App"
set   :deploy_to,     "/var/www/my-app.com"
set   :domain,        "my-app.com"

set   :scm,           :git
set   :repository,    "ssh-gitrepo-domain.com:/path/to/repo.git"

role  :web,           domain
role  :app,           domain, :primary => true

set   :use_sudo,      false
set   :keep_releases, 3

{% endhighlight %}

在这种情况下，运行 cap deploy, capifony 会进行以下步骤:

* ssh 登录产线服务器 (my-app.com)
* 创建一个新的版本目录 (/var/www/my-app.com/releases/...)
* 从git版本库 clone 最新的项目版本 (ssh-gitrepo-domain.com)
* 复制代码。pull git 版本到新版本目录
* 运行发布钩子(cache:warmup, cc, 等。)

**注意**：默认情况下 capifony 不会清除旧版本，你必须设置 keep_releases 参数，并且在 deploy.rb 中添加任务来使其运行，如： after "deploy", "deploy:cleanup" 。

如果你不想每次都 clone 整个版本库，你可以设置 :deploy_via 参数：

{% highlight ruby %}
set :deploy_via, :remote_cache
{% endhighlight %}

这样，服务端会保存一个git版本库，而 Capifony 只会 fetch 上一次发布以来的修改。

b) deployment → production (通过复制)

第二种方式通过复制开发机到产线来发布。这种情况下，开发机（可能是你的本地电脑）需要有访问git版本库的权限，并且可以执行pull操作。

由于产线服务器也需要安装vendors，因此如果你的vendors使用远程的私有源，那么产线服务器也必须拥有这些源的pull权限，否则 composer 或者 bin/vendors 安装时会失败。如果你想在发布前在本地安装 vendors ，请参考下面的手册：[Install vendors locally before deploy](http://capifony.org/cookbook/copy-deps-locally-strategy.html)。

使用此方式时发布服务器也必须可以 ssh 访问产线服务器：

{% highlight ruby %}

# deploy.rb

set   :application,   "My App"
set   :deploy_to,     "/var/www/my-app.com"
set   :domain,        "my-app.com"

set   :scm,           :git
set   :repository,    "file:///Users/deployer/sites/my-app"
set   :deploy_via,    :copy

role  :web,           domain
role  :app,           domain, :primary => true

set   :use_sudo,      false
set   :keep_releases, 3

{% endhighlight %}

在这种情况下，运行 cap deploy, capifony 会进行以下步骤:

* ssh 登录产线服务器 (my-app.com)
* 创建一个新的版本目录 (/var/www/my-app.com/releases/...)
* 从本地git版本库 clone 最新的项目版本
* 复制代码。pull git 版本到产线服务器
* 运行发布钩子(cache:warmup, cc, 等。)

当然，每次发布都要复制整个项目，是昂贵而且缓慢的，幸好你可以通过 capistrano_rsync_with_remote_cache gem 来优化流程:

{% highlight ruby %}
gem install capistrano_rsync_with_remote_cache
{% endhighlight %}

然后，在 deploy.rb 中修改发布流程:

{% highlight ruby %}
set :deploy_via, :rsync_with_remote_cache
{% endhighlight %}

这样，rsync会在你的产线服务器上生成缓存，并且只提交两次发布简修改的文件。

**4. 配置服务器**

现在你可以开始发布了！cd 到你的本地项目目录，运行下面命令来使产线服务器设置好 Capistrano 要求的目录结构：

{% highlight shell %}
cap deploy:setup
{% endhighlight %}

( 你只需要运行这个命令一次 )

此命令会在你的服务器上创建类似的目录结构。详细的结构会根据你发布的是 symfony 1.x 还是 symfony 2 而有所不同：

{% highlight shell %}
`-- /var/www/my-app.com
|-- current → /var/www/my-app.com/releases/20100512131539
|-- releases
|   `-- 20100512131539
|   `-- 20100509150741
|   `-- 20100509145325
`-- shared
   |-- web
   |    `-- uploads
   |-- log
   `-- config
       `-- databases.yml
{% endhighlight %}

release 目录下的是实际的发布代码，以时间戳来命名。以 symfony 1.x 为例，Capistrano 链接 app 中的 log 与 web/uploads 目录到一个共享目录，这样你发布一个新版本后这些内容不会被清除。

你可以使用以下命令来快速设置一个新服务器：

{% highlight shell %}
cap HOSTS=new.server.com deploy:setup
{% endhighlight %}

**5. 发布**

运行下面命令来发布：

{% highlight shell %}
cap deploy
{% endhighlight %}

根据配置的不同，你可能需要 ssh 登录到你的服务器来进行一些附加的配置，在第一次发布后共享文件（如 app/config/parameters.yml，如果你是使用下面的方式来发布 Symfony2 的话）。


{% highlight shell %}
Something went wrong???
cap deploy:rollback
{% endhighlight %}

**Symfony2 发布**

如果你需要发布 Symfony2 应用，那么此部分内容是为你而设的。本节介绍如何配置 capifony 来发布一个使用 bin/vendors 来管理 vendor 库，并且使用 app/config/parameters.yml 来进行服务器相关配置（如数据库连接信息）的应用。

首先，在 app/config/deploy.rb 添加以下内容，设置 parameters.yml 在不同的发布版本简时共享的：

{% highlight ruby %}
set :shared_files,      ["app/config/parameters.yml"]
{% endhighlight %}

然后，在不同发布简共享 vendor 目录，这样发布起来会快一点：

{% highlight ruby %}
set :shared_children,     [app_path + "/logs", web_path + "/uploads", "vendor"]
{% endhighlight %}

capifony 默认情况下依赖于 bin/verdors 来安装第三方库。然而现在默认的依赖管理工具时 Composer，你可以使用下面的配置来使用：
{% highlight ruby %}
set :use_composer, true
{% endhighlight %}

如果你想升级第三方包，请配置以下参数:

{% highlight ruby %}
set :update_vendors, true
{% endhighlight %}

这样它会运行 composer.phar（如果你使用Composer） 或者 bin/vendors。需要注意的时 bin/vendors 可以通过配置 :vendors_mode 参数来指定运行哪个操作(upgrade, install, 或者 reinstall)。

最后一步时配置你的 app/config/parameters.yml 文件。最好的方式是手动在服务器的共享目录中创建：

{% highlight shell %}
ssh your_deploy_server
mkdir -p /var/www/my-app.com/shared/app/config
vim /var/www/my-app.com/shared/app/config/parameters.yml
{% endhighlight %}

parameters.yml文件正确配置后，你应该可以测试你发布的应用。每次新的发布后，同一个 app/config/parameters.yml 会链接到你的应用，因此你只需要在初次发布时配置它。

**有用的任务**

如果你需要发布并且运行迁移命令，运行：

{% highlight shell %}
cap deploy:migrations
{% endhighlight %}

运行以下命令，在产线运行测试:

{% highlight shell %}
cap deploy:test_all
{% endhighlight %}

通过下面命令来运行任务/命令:

{% highlight shell %}
cap symfony
{% endhighlight %}

通过下面命令获取所有可用任务的信息:

{% highlight shell %}
cap -vT
{% endhighlight %}


**Capifony 自动重启Apache**

如果你的服务器安装了APC，你会发现每次 Deploy 完后都需要重启 Apache（php-fpm）来使新代码生效。
Capifony 提供了重启的任务，你只需要进行覆写：

{% highlight ruby %}

# Custom(ised) tasks
namespace :deploy do
	# Apache needs to be restarted to make sure that the APC cache is cleared.
	# This overwrites the :restart task in the parent config which is empty.
	desc "Restart Apache"
	task :restart, :except => { :no_release => true }, :roles => :app do
		run "sudo /etc/init.d/php5-fpm restart"
		puts "--> PHP successfully restarted".green
	end
end

{% endhighlight %}

**更新：Symfony3 发布**


现时（2016-05）Capifony只支持Symfony 1或者Symfony 2的发布，而由于Symfony 3的更改并不大，可以通过修改配置的方式让其兼容。

Symfony 3主要的更改在于原来位于app目录中的console，logs，cache等目录，现在变成了更加规范的结构，console在bin目录下，而其他几个可写目录则存放在独立的var目录

对 deploy.rb 配置文件进行以下修改

{% highlight ruby %}
set :symfony_console, "bin/console"
set :cache_path, "var/cache"
set :log_path, "var/logs"


set :shared_children,     ["var/logs", web_path + "/files", web_path + "/uploads", "vendor", "var/sessions", "var/cache"]
set :writable_dirs,       ["var/cache", "var/logs", "var/sessions"]
{% endhighlight %}

以上具体的项根据项目的不同可能有所区别。


