---
title:  "使用Nginx做AWS S3的反向代理"
categories: Linux Nginx AWS
---

对于一些UGC（用户创建内容）类网站而言，特别是图片网站，随着用户数的增长，时间的推移，网站上的文件会越来越多；得益于云服务的出现，存储系统的扩展变得简单，一个比较常用的做法就是将文件存储于AWS S3，然后用户通过S3或者CloudFront下载。然而虽然AWS S3的储存价格相对便宜，但流量价格却非常高，最终导致网站的托管开销增加。

下面介绍的方式，文件存储在AWS S3服务器上，Nginx配置成S3的反向代理，Nginx将热门文件缓存在本地服务器，大大减少对S3的文件请求，从而降低流量费用。

```
proxy_cache_path  /var/nginx/cache/aws  levels=2:2:2 use_temp_path=off keys_zone=aws:1000m inactive=30d max_size=100g;

server{
        ...
        location @proxy {
                set $s3_bucket        'testaws.s3.amazonaws.com';
                add_header x-by "aws";

                proxy_http_version     1.1;
                proxy_set_header       Host $s3_bucket;
                proxy_set_header       Authorization '';
                proxy_hide_header      x-amz-id-2;
                proxy_hide_header      x-amz-request-id;
                proxy_hide_header      Set-Cookie;
                proxy_ignore_headers   "Set-Cookie";
                proxy_buffering        on;
                proxy_intercept_errors on;

                proxy_cache            aws;
                proxy_cache_valid      any 1m;
                proxy_cache_valid      200 302 30d;
                proxy_cache_bypass     $http_cache_purge;
                add_header             X-Cached $upstream_cache_status;
                proxy_cache_lock on;
                proxy_buffer_size 128k;
                proxy_buffers 200 128k;


                resolver               8.8.8.8 valid=300s;
                resolver_timeout       10s;

                proxy_pass             http://$s3_bucket$uri;
       }
       ...
}
```

使用上面的方式对一台Linode 16G服务器进行配置，原来文件均存储在本地，本地磁盘空间使用接近90%，CPU与带宽余量还较大，如果对Linode进行升级，则成本会增加一倍，并且随着内容的增加，不久的未来又会因为磁盘问题需要升级到更高版本；使用上面方案，先将旧附件迁移到AWS S3，可以满足文件持续增长的需求，服务器方面的开销也会跟网站的发展比较同步，不会出现跳跃式的增加。
