---
title:  "ELK日志分析入门"
categories: ELK
published: false
---

ELK Stack 由 Elastic.co 公司名下的 Elasticsearch、Logstash、Kibana 三个开源软件的组成，用于日志的快速搜索和分析。

![ELK stack]({{ site.url }}/assets/2016/elk.png "ELK stack")

本文整理了 ELK 的相关基础知识，并且结合最近对nginx日志分析的一个需求进行记录。

* TOC
{:toc}

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

公司的资讯网站运行在一个相对老旧的CMS上，虽然做了不少努力，但依然还有不少功能是直接通过主域名来获取文件、附件等静态资源

## ELK 安装

## Logstash

## Elasticsearch

## Kibana


## 参考资料

* [The Complete Guide to the ELK Stack](http://logz.io/learn/complete-guide-elk-stack/)
* [ELKstack 中文指南](http://kibana.logstash.es/content/)
