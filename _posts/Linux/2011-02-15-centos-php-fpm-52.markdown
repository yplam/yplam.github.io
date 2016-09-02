---
title:  "Centos Nginx PHP-FPM Mysql 服务器环境配置"
categories: Linux
---

此笔记基于Linode Centos 5.x 64 bit 系统，安装与配置LNMP服务器环境，此配置主要用于运行Drupal。 

安装Mysql：

{% highlight shell %}

yum install mysql-server mysql-devel

{% endhighlight %}

安装编译库： 

{% highlight shell %}

make gcc patch flex bison autoconf libjpeg libjpeg-devel libpng libpng-devel gd gd-devel libxml2 libxml2-devel zlib zlib-devel glib2 glib2-devel bzip2 bzip2-devel libevent libevent-devel ncurses ncurses-devel curl curl-devel openssl openssl-devel libmcrypt libmcrypt-devel mhash-devel libxslt-devel pcre-devel

{% endhighlight %}

下载、编译、安装PHP-FPM 

{% highlight shell %}

    wget http://us.php.net/distributions/php-5.2.17.tar.gz
    tar -xvzf php-5.2.17.tar.gz
    wget http://php-fpm.org/downloads/php-5.2.17-fpm-0.5.14.diff.gz
    gzip -cd php-5.2.17-fpm-0.5.14.diff.gz | patch -d php-5.2.17 -p1
    cd php-5.2.17

    
#对于64位系统注意要指定--with-libdir=lib64

    ./configure --enable-fastcgi --enable-fpm --with-mcrypt --with-zlib --enable-mbstring --enable-pdo --with-mysql --with-mysqli --with-curl --disable-debug --with-pic --disable-rpath --enable-inline-optimization --with-bz2 --enable-xml --with-zlib --enable-sockets --enable-sysvsem --enable-sysvshm --enable-pcntl --enable-mbregex --with-mhash --with-xsl --enable-zip --with-pcre-regex --without-pdo-sqlite --with-pdo-mysql --without-sqlite --with-jpeg-dir --with-png-dir --with-gd --with-openssl --with-libdir=lib64
    
    make 
    make install 
    strip /usr/local/bin/php-cgi 
    cp sapi/cgi/fpm/php-fpm /etc/init.d/
    chmod +x /etc/init.d/php-fpm
    

    wget --no-check-certificate https://github.com/downloads/indeyets/syck/syck-0.70.tar.gz
#编译安装syck-0.70（略）
    
    pecl install memcache
    pecl install apc
    pecl install syck-beta
    
    cp php.ini-recommended /usr/local/lib/php.ini
{% endhighlight %}

更改php-fpm.conf配置，设置运行用户。位置大概在51、52、63、66行。 

{% highlight xml %}
    <value name="owner">www</value>
    <value name="group">www</value>
    <value name="user">www</value>
    <value name="group">www</value>
{% endhighlight %}

 安装nginx 
 
{% highlight shell %}
    wget ...nginx
    ./configure --sbin-path=/usr/local/sbin --with-http_ssl_module --user=www --group=www --with-http_gzip_static_module
    make
    make install
{% endhighlight %}

 nginx 与 drupal 的配置可以参考 github项目 <https://github.com/yhager/nginx_drupal>
