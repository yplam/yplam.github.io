---
title:  "为个人博客配置 HTTPS 访问"
categories: Linux Nginx
---

如何能让个人博客显得更有逼格？配置成HTTPS访问是一个不错的选择，下面分享 yplam.com 切换到 HTTPS 的过程。

**环境**

yplam.com 域名注册商为Godaddy，托管商为Linode，使用 debian + nginx + php 的服务器环境

**配置过程**

首先，需要购买个SSL证书，当然这不是免费的，甚至有可能很贵。在这里我们使用Godaddy最便宜的SSL证书，每年四百多，使用优惠码还可以便宜30%。

购买证书时需要提供csr文件内容，这个可以在服务器提前生成：

```
openssl req -new -newkey rsa:2048 -nodes -keyout yplam.key -out yplam.csr

```

然后根据你的需求填入信息，其中比较重要的是 Common Name 需要填你证书的域名。

生成后打开csr文件，将内容拷贝到Godaddy SSL证书申请的输入框中，提交。

如无意外，付款后就可以进入配置页面（在这过程之间可能需要等待几分钟）

因为域名也是在Godaddy上注册的，接着不需要做其他配置，只需要等待过程完成。

完成后Godaddy会让你下载一个zip文件（web服务器类型选其他），将zip上传到服务器，用unzip命令解压出来两个 .crt 文件，其中以 gd_bundle 开头的文件为链式证书，另一个为服务器证书，使用nginx时需要将其合为一个：

```
cat e1bb80d***.crt gd_bundle-***.crt > yplam.crt

```

需要注意的是合并时的顺序是服务器证书在前，否则nginx会报错：

```
SSL_CTX_use_PrivateKey_file(" ... /www.example.com.key") failed
   (SSL: error:0B080074:x509 certificate routines:
    X509_check_private_key:key values mismatch)

```

然后可以配置服务器：

```

server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.chained.crt;
    ssl_certificate_key www.example.com.key;
    ...
}

```

配置完成，测试是否正常：

```

 openssl s_client -connect www.godaddy.com:443

```

最后，如果你设置整站HTTPS的话不要忘了在nginx中对旧内容做个跳转：

```

server {
    listen       80;
    server_name  www.yplam.com yplam.com;
    return       301 https://yplam.com$request_uri;
}


```

一切配置完成后，可以访问 https://www.ssllabs.com/ssltest/index.html 对服务器的配置情况进行校验，然后根据相关建议选择进行修复

譬如，在debian 8 下会有 Diffie-Hellman (DH) key 警告，那么可以参考 https://weakdh.org/sysadmin.html 这里的提示进一步进行安全配置。

**参考文档**

http://nginx.org/en/docs/http/configuring_https_servers.html

https://www.godaddy.com/help/generating-nginx-csrs-certificate-signing-requests-3601
