---
title:  "ELK日志分析入门"
categories: ELK
---

ELK Stack 由 Elastic.co 公司名下的 Elasticsearch、Logstash、Kibana 三个开源软件的组成，用于日志的快速搜索和分析。

![ELK stack]({{ site.url }}/assets/2016/elk.png "ELK stack")

本文整理了 ELK 的相关基础知识，并且结合最近对nginx日志分析的一个需求进行记录。

## ELK 简介

ELK Stack 是 Elasticsearch、Logstash、Kibana 三个开源软件的组合。在实时数据检索和分析场合，三者通常是配合共用，而且又都先后归于 Elastic.co 公司名下，故有此简称。

ELK Stack 在最近两年迅速崛起，成为机器数据分析，或者说实时日志处理领域，开源界的第一选择。和传统的日志处理方案相比，ELK Stack 具有如下几个优点：

* 处理方式灵活。Elasticsearch 是实时全文索引，不需要像 storm 那样预先编程才能使用；
* 配置简易上手。Elasticsearch 全部采用 JSON 接口，Logstash 是 Ruby DSL 设计，都是目前业界最通用的配置语法设计；
* 检索性能高效。虽然每次查询都是实时计算，但是优秀的设计和实现基本可以达到全天数据查询的秒级响应；
* 集群线性扩展。不管是 Elasticsearch 集群还是 Logstash 集群都是可以线性扩展的；
* 前端操作炫丽。Kibana 界面上，只需要点击鼠标，就可以完成搜索、聚合功能，生成炫丽的仪表板。

摘自 [ELKstack 中文指南](http://kibana.logstash.es/content/)

## 使用ELK的原因

公司的资讯网站运行在一个相对老旧的CMS上，虽然有记录nginx访问日志，但通常是用于数据后分析上，对服务器突发的CPU、带宽暴涨，缺乏快速的分析响应。使用ELK后可以很直观的在Kibana面板
上看到关于网站访问的实时数据，并且可以针对重点需求设置特定面板，通过快速查看实时日志分析各种服务器资源波动原因。

## ELK 安装

ELK的安装相对简单，只需要安装好java，然后添加官方源就可以使用系统自带的包管理工具安装。

详细安装过程可以参考下面两个链接 [ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-14-04), [Centos](https://developers.redhat.com/blog/2016/06/07/how-to-install-elastic-stack-elk-on-red-hat-enterprise-linux-rhel/)

## Logstash

Logstash 可以看作是一个ETL工具，负责对日志的抽取、转换、加载，在ELK栈中，Logstash 负责监听日志文件，从日志文件中抽取有用数据，拆分，转换，然后保存到 Elasticsearch。Logstash 除了可以监听日志文件以外，
还可以跟filebeat等工具配合使用，将多个服务器的日志集合到一起，再中转存储。Logstash 会将日志按天来生成 Type，添加到索引中。

```
input {
    beats {
		host => "1.2.3.4"
        port => 5044
        ssl => false
        codec => json
    }
} 
filter { 
    mutate { 
        convert => ["status", "integer"] 
    } 
    mutate { 
        remove_field => ["@version"] 
    } 
}
output { 
    elasticsearch { 
        hosts => ["1.2.3.4:9200"] 
        index => "abc-log" 
        document_type => "%{+YYYY.MM.dd}" 
        workers => 1 
        flush_size => 200 
        idle_flush_time => 5 
    } 
}

```

## Elasticsearch

在Logstash发送日志信息过来之前，Elasticsearch 需要先创建好索引mapping。

```
{
  "mappings": {
    "_default_": {
      "dynamic_templates": [
        {
          "notanalyzed": {
            "mapping": {
              "index": "not_analyzed",
              "type": "string"
            },
            "match": "*",
            "match_mapping_type": "string"
          }
        }
      ],
      "properties": {
        "agent": {
          "type": "string"
        },
        "clientip": {
          "index": "not_analyzed",
          "type": "string"
        },
        "host": {
          "index": "not_analyzed",
          "type": "string"
        },
        "http_host": {
          "index": "not_analyzed",
          "type": "string"
        },
        "http_referer": {
          "index": "not_analyzed",
          "type": "string"
        },
        "path": {
          "index": "not_analyzed",
          "type": "string"
        },
        "request": {
          "type": "string"
        },
        "responsetime": {
          "type": "double"
        },
        "size": {
          "type": "long"
        },
        "status": {
          "type": "long"
        }
      }
    }
  }
}

```

## Kibana

Kibana 不需要太多的配置，只需要访问到 Elasticsearch，并且发现相关的索引。

Kibana 按照 搜索 -> 可视化 -> 面板 的流程，将你需要看到的信息逐步构建出来。

## 使用phantomjs定期发送Kibana报告


下面是使用 phantomjs 发送Kibana报告的测试代码。其原理是使用 phantomjs 模拟浏览器访问 Kibana，然后对访问内容进行截图，然后将生成的截图通过 http post的方式调用 sendcloud 接口发送邮件。（发送邮件的调用有点繁琐，因为找不到很好的读取本地文件的方式，而 phantomjs webpage 模块的 renderBuffer 实际上又无法使用）


```

var page = require('webpage').create();
var fs = require('fs');
var url = "***";
page.settings.userName = "***"; // http auth
page.settings.password = "***";
page.viewportSize = { width: 1280, height: 900 };
page.open(url, function(status) {
  if (status !== 'success') {
    console.log('Unable to load the address!');
    phantom.exit(1);
  } else {
    window.setTimeout(function () {
        page.render('abc.png');
        var rawFile = new XMLHttpRequest();
        rawFile.open("GET", 'file://****abc.png', false);
        rawFile.responseType = "blob";
        rawFile.onload = function () {
          var formPost = new FormData();
          var apiUser = '***'; 
          var apiKey = '***';
          var currentDate = new Date();
          formPost.append('apiUser', apiUser);
          formPost.append('apiKey', apiKey);
          formPost.append('from', '***');
          formPost.append('to', '***');
          formPost.append('subject', '***数据日报 - '+currentDate.toString());
          formPost.append('html', '<h1>***数据日报</h1><p>此数据由程序自动生成，如果需要调整相关指标，请反馈给产品经理。</p><p><img src="cid:abc.png"/></p>');
          formPost.append('useAddressList', true);
          formPost.append('attachments', rawFile.response, 'abc.png');
          formPost.append('embeddedImage', rawFile.response, 'abc.png');
          formPost.append('embeddedCid', "abc.png");
          var xhr = new XMLHttpRequest();
          xhr.open('POST', 'http://api.sendcloud.net/apiv2/mail/send', true);
          xhr.send(formPost);
          xhr.onload = function (e) {
            if (xhr.readyState === 4) {
              if (xhr.status === 200) {
                console.log(xhr.responseText);
              } else {
                console.error(xhr.statusText);
              }
            }
          };
          xhr.onerror = function (e) {
            console.error(xhr.statusText);
          };
        };
        rawFile.send(null);

        
        window.setTimeout(function(){
            phantom.exit();
        }, 5000);
    }, 20000);
  }
});


```


将上面内容保存为js文件，然后用下面命令调用

```

phantomjs --web-security=false --local-to-remote-url-access=true ***.js

```

### 参考资料

* [The Complete Guide to the ELK Stack](http://logz.io/learn/complete-guide-elk-stack/)
* [ELKstack 中文指南](http://kibana.logstash.es/content/)
