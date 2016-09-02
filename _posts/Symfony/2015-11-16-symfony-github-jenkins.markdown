---
title:  "使用 Jenkins 与 Github 对 Symfony 2 应用进行持续集成"
categories: PHP Symfony
---

持续集成是一种软件开发实践，对于提高软件开发效率并保障软件开发质量提供了理论基础。Jenkins 是一个开源软件项目，旨在提供一个开放易用的软件平台，使持续集成变成可能。本文记录如何使用 Jenkins 与 Github 对 Symfony 2 应用进行持续集成。

**服务器环境**

下面的内容基于Debian 7, Linode 2GB, PHP 5.5, Nginx 1.8, 理论上应该适用于最近版本的Debian、Ubuntu。

### 持续集成概述

**什么是持续集成**

随着软件开发复杂度的不断提高，团队开发成员间如何更好地协同工作以确保软件开发的质量已经慢慢成为开发过程中不可回避的问题。尤其是近些年来，敏捷（Agile） 在软件工程领域越来越红火，如何能再不断变化的需求中快速适应和保证软件的质量也显得尤其的重要。

持续集成正是针对这一类问题的一种软件开发实践。它倡导团队开发成员必须经常集成他们的工作，甚至每天都可能发生多次集成。而每次的集成都是通过自动化的构建来验证，包括自动编译、发布和测试，从而尽快地发现集成错误，让团队能够更快的开发内聚的软件。

持续集成的核心价值在于：

* 持续集成中的任何一个环节都是自动完成的，无需太多的人工干预，有利于减少重复过程以节省时间、费用和工作量；
* 持续集成保障了每个时间点上团队成员提交的代码是能成功集成的。换言之，任何时间点都能第一时间发现软件的集成问题，使任意时间发布可部署的软件成为了可能；
* 持续集成还能利于软件本身的发展趋势，这点在需求不明确或是频繁性变更的情景中尤其重要，持续集成的质量能帮助团队进行有效决策，同时建立团队对开发产品的信心。

**持续集成的原则**

业界普遍认同的持续集成的原则包括：

* 需要版本控制软件保障团队成员提交的代码不会导致集成失败。常用的版本控制软件有 IBM Rational ClearCase、CVS、Subversion 等；
* 开发人员必须及时向版本控制库中提交代码，也必须经常性地从版本控制库中更新代码到本地；
* 需要有专门的集成服务器来执行集成构建。根据项目的具体实际，集成构建可以被软件的修改来直接触发，也可以定时启动，如每半个小时构建一次；
* 必须保证构建的成功。如果构建失败，修复构建过程中的错误是优先级最高的工作。一旦修复，需要手动启动一次构建。

**持续集成系统的组成**

由此可见，一个完整的构建系统必须包括：

* 一个自动构建过程，包括自动编译、分发、部署和测试等。
* 一个代码存储库，即需要版本控制软件来保障代码的可维护性，同时作为构建过程的素材库。
* 一个持续集成服务器。本文中介绍的 Jenkins 就是一个配置简单和使用方便的持续集成服务器。

以上内容引用自 [https://www.ibm.com/developerworks/cn/java/j-lo-jenkins/](https://www.ibm.com/developerworks/cn/java/j-lo-jenkins/)

### 安装Jenkins

Jenkins 是一个开源项目，提供了一种易于使用的持续集成系统，使开发者从繁杂的集成中解脱出来，专注于更为重要的业务逻辑实现上。同时 Jenkins 能实施监控集成中存在的错误，提供详细的日志文件和提醒功能，还能用图表的形式形象地展示项目构建的趋势和稳定性。

Debian 下 Jenkins 的安装非常简单

```
wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

安装完成后，可以通过 /etc/default/jenkins 文件修改jenkins的启动参数；然后启动服务

```
service jenkins start
```

也可以将jenkins加入到自动启动服务中

```
update-rc.d jenkins defaults
```

在默认配置下，访问服务器的8080端口，就可以看到Jenkins的界面；当然，你的服务器可能只开了nginx的80端口，那么需要配置nginx反向代理，以下是一个简单的配置：

```
server {

    listen 80;
    server_name ****.com; # 你的服务器域名

    location / {

      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      proxy_pass          http://127.0.0.1:8080;
      proxy_read_timeout  90;

    }
}

```

这样你就可以通过 80 端口访问jenkins。

安装完Jenkins后你会发现，Jenkins默认没有开启任何安全认证，所以第一步就是在 Manage Jenkins > Configure Global Security 中勾选 Enable security，使用 Jenkins’ own user database，Allow users to sign up，Logged-in users can do anything；然后注册、登录一个账号；再取消 Allow users to sign up选项；这里的操作有点繁琐，目的只是为了在你配置好Jenkins之前不会被别人捣乱。

**配置插件与PHP库**


在 Manage Jenkins > Manage Plugin 的 Available 标签中搜索安装以下插件

```
CheckStyle
Clover PHP
HTML Publisher
DRY
JDepend
PMD
Violations
xUnit
Git
Github
Green Balls
```

然后安装对应的PHP库，可以手动下载或者用 PEAR :

```
pear channel-discover pear.pdepend.org
pear channel-discover pear.phpmd.org
pear channel-discover pear.phpunit.de
pear channel-discover pear.phpdoc.org
pear channel-discover components.ez.no
pear channel-discover pear.symfony-project.com

pear install pdepend/PHP_Depend
pear install phpmd/PHP_PMD
pear install phpunit/phpcpd
pear install phpunit/phploc
pear install --alldeps phpunit/PHP_CodeBrowser
pear install --alldeps phpunit/PHPUnit
pear install phpdoc/phpDocumentor-alpha
pear install PHP_CodeSniffer

```

对于Symfony2 项目，我们还需要安装Symfony codeing standard，参考：[https://github.com/escapestudios/Symfony2-coding-standard](https://github.com/escapestudios/Symfony2-coding-standard)

**配置ANT BUILD**

在代码的根目录添加build.xml文件：

```
<?xml version="1.0" encoding="UTF-8"?>
<project name="name-of-project" default="build-local">
    <property name="workspace" value="${basedir}" />
    <property name="sourcedir" value="${basedir}/src" />
    <property name="builddir" value="${workspace}/app/build" />

    <target name="build" depends="prepare,parameters,vendors,lint,phploc,pdepend,phpmd-ci,phpcs-ci,phpcpd,phpunit,phpcb"/>
    <target name="build-local" depends="prepare,lint,phploc,pdepend,phpmd-ci,phpcs-ci,phpcpd,phpunit"/>

    <target name="clean" description="Cleanup build artifacts">
        <delete dir="${basedir}/app/build/api"/>
        <delete dir="${basedir}/app/build/code-browser"/>
        <delete dir="${basedir}/app/build/coverage"/>
        <delete dir="${basedir}/app/build/logs"/>
        <delete dir="${basedir}/app/build/pdepend"/>
    </target>

    <target name="prepare" depends="clean" description="Prepare for build">
        <mkdir dir="${basedir}/app/build/api"/>
        <mkdir dir="${basedir}/app/build/code-browser"/>
        <mkdir dir="${basedir}/app/build/coverage"/>
        <mkdir dir="${basedir}/app/build/logs"/>
        <mkdir dir="${basedir}/app/build/pdepend"/>
        <mkdir dir="${basedir}/app/build/phpdox"/>
    </target>

    <target name="phpunit" description="Run unit tests with PHPUnit">
        <exec executable="phpunit" failonerror="true">
            <arg value="-c" />
            <arg path="app" />
        </exec>
    </target>

    <target name="lint" description="Perform syntax check of sourcecode files">
        <apply executable="php" failonerror="true">
            <arg value="-l" />
            <fileset dir="${sourcedir}">
                <include name="**/*.php" />
                <modified />
            </fileset>
            <fileset dir="${basedir}/src/">
                <include name="**/*Test.php" />
                <modified />
            </fileset>
        </apply>
    </target>

    <target name="phploc" description="Measure project size using PHPLOC">
        <exec executable="phploc">
            <arg value="--log-csv" />
            <arg value="${basedir}/app/build/logs/phploc.csv" />
            <arg path="${basedir}/src" />
        </exec>
    </target>

    <target name="pdepend" description="Calculate software metrics using PHP_Depend">
        <exec executable="pdepend">
            <arg value="--jdepend-xml=${basedir}/app/build/logs/jdepend.xml" />
            <arg value="--jdepend-chart=${basedir}/app/build/pdepend/dependencies.svg" />
            <arg value="--overview-pyramid=${basedir}/app/build/pdepend/overview-pyramid.svg" />
            <arg path="${basedir}/src" />
        </exec>
    </target>

    <target name="phpmd-ci" description="Perform project mess detection using PHPMD creating a log file for the continuous integration server">
        <exec executable="phpmd">
            <arg path="${basedir}/src" />
            <arg value="xml" />
            <arg value="${basedir}/app/Resources/jenkins/phpmd.xml" />
            <arg value="--reportfile" />
            <arg value="${basedir}/app/build/logs/pmd.xml" />
        </exec>
    </target>

    <target name="phpcs-ci" description="Find coding standard violations using PHP_CodeSniffer creating a log file for the continuous integration server">
        <exec executable="phpcs">
            <arg value="--report=checkstyle" />
            <arg value="--report-file=${basedir}/app/build/logs/checkstyle.xml" />
            <arg value="--standard=Symfony2" />
            <arg path="${basedir}/src" />
        </exec>
    </target>

    <target name="phpcpd" description="Find duplicate code using PHPCPD">
        <exec executable="phpcpd">
            <arg value="--log-pmd" />
            <arg value="${basedir}/app/build/logs/pmd-cpd.xml" />
            <arg path="${basedir}/src" />
        </exec>
    </target>

    <target name="phpcb" description="Aggregate tool output with PHP_CodeBrowser">
        <exec executable="phpcb">
            <arg value="--log" />
            <arg path="${basedir}/app/build/logs" />
            <arg value="--source" />
            <arg path="${basedir}/src" />
            <arg value="--output" />
            <arg path="${basedir}/app/build/code-browser" />
        </exec>
    </target>


    <target name="vendors" description="Update vendors">
        <exec executable="composer" failonerror="true">
            <arg value="update" />
        </exec>
    </target>

    <target name="parameters" description="Copy parameters">
        <exec executable="cp" failonerror="true">
            <arg path="app/config/parameters.yml.dist" />
            <arg path="app/config/parameters.yml" />
        </exec>
    </target>

</project>
```

添加 app/Resources/jenkins/phpmd.xml 文件：

```

<?xml version="1.0"?>
<ruleset name="Symfony2 ruleset" xmlns="http://pmd.sf.net/ruleset/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://pmd.sf.net/ruleset/1.0.0 http://pmd.sf.net/ruleset_xml_schema.xsd" xsi:noNamespaceSchemaLocation="http://pmd.sf.net/ruleset_xml_schema.xsd">
    <description>
        Custom ruleset.
    </description>

    <rule ref="rulesets/design.xml" />
    <rule ref="rulesets/unusedcode.xml" />
    <rule ref="rulesets/codesize.xml" />
    <rule ref="rulesets/naming.xml" />

</ruleset>

```

以上的配置仅作为参考，您可以根据自己的项目有所增删；需要注意的是，上面配置中，要求 app/config/parameters.yml.dist 中的内容为测试机上面的环境配置，譬如：当你的phpunit中需要跟数据库有交互，这些配置选项必须正确。

**配置Github**

在开始创建Jenkins工程前，需要先确保你的服务器能正常访问Github。

* 点击 Manage Jenkins > Configure System 找到 GitHub Plugin Configuration，选择Add github server config
* 选择下边的 Advanced... 在 Additional actions 中选择cover login and password to token，输入github的用户名与密码，点击create token，会生成一个token供以后使用（这个可以在github后台进行管理）
* 由于新生成的token不会自动刷新，在此需要先保存，然后重新进入此配置页面，这样在github配置的Credentials选项里就会多了上面生成的token
* 选择该token，然后点击validate credentials，会返回验证成功的信息

完成上面步骤后Jenkins就有权限访问你的github项目，譬如Jenkins可以自动更新项目的Webhooks，而无需在Github网站进行手动操作。

另外，如果你使用的时私有工程，那么你需要为Jenkins用户配置访问github的ssh keys，详情请参考：

**创建Jenkins工程**

* 点击 Create New Job，选择 Free Style Project
* 配置github相关地址
* 选择 Build when a change is pushed to GitHub
* Build类型选择ANT，地址为build.xml
* 根据需要添加POST Action，如 CheckStype  **/app/build/logs/checkstyle.xml
* pmd  **/app/build/logs/pmd.xml
* Duplicate code results **/app/build/logs/cpd.xml

**运行**

配置完成后，点击Build Now开始运行，此时可以在该任务页面中打开终端查看运行状态，进行调错。如无意外，你每次push到github后就会自动触发Jenkins对项目进行Build。

**安全配置**

至此，Jenkins已经可以正常运行，然而，你会发现：就算是没有登录的用户，都可以随意的查看你网站上的项目概况，这对于私有项目来说很不好，因此我们需要配置基于角色的权限管理。

* 安装 Role-based Authorization Strategy 插件
* 在安全配置处启用 Role-Based Strategy
* Manage Jenkins > Manage and Assign Roles 中按需进行配置，如果只有一个用户，也可以使用默认配置

这样，没登录用户将无法查看关于Jenkins的项目信息。

**参考资料**

http://blog.lazycloud.net/en/using-jenkins-for-a-symfony2-project/
