---
title:  "配置 Nginx Http 认证以及 IP 访问策略"
categories: Linux Nginx
---

网络世界充满漏洞，同样，网络世界充满着寻找漏洞的人。为Web服务器的后台路径添加HTTP AUTH认证，是防范漏洞被探测到的一个简单有有效的方法。

不妨打开你网站的404日志，其中肯定不乏一堆对后台路径的试探性探测，这其中有一些是用来确定你网站使用的是哪种系统，而更多的是直接对各种已知系统漏洞的扫描。

一个比较有效阻止这种扫描的方法是在Web服务器的层面上添加HTTP认证。

首先，安装htpasswd工具

```
sudo apt-get install apache2-utils
```

然后生成用户名密码：

```
sudo htpasswd -c /etc/nginx/.htpasswd myuser
```

最后，为需要进行保护的路径添加认证

```
location location ~ /admin(/.*)  {
      ...
      auth_basic "Restricted";                                
      auth_basic_user_file /etc/nginx/.htpasswd; 
      ...
  }
```

这样，用户在访问admin路径下的内容时都会多一层HTTP认证。

然而，我们可以对上面的配置做一点人性化的修改。譬如，针对公司用户而言，IP地址是固定的，我们可以将该地址设为可信，这样就免除了公司内部人员认证的麻烦：

```
location location ~ /admin(/.*)  {
      ...
      satisfy any;
      include allow-ips.conf;
      deny  all;

      auth_basic "Restricted";                                
      auth_basic_user_file /etc/nginx/.htpasswd; 
      ...
  }
```

allow-ips.conf 配置免除HTTP认证的IP地址，内容如：

```
allow 119.147.176.159;
```
