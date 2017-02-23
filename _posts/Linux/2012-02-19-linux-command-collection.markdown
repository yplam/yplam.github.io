---
title:  "[持续更新] Linux 下的一些命令技巧"
categories: Linux
---

此笔记记录 Linux 下的一些命令使用技巧，作为工作中的备忘。


### 导入数据库时跳过某个表

有时我们需要将某天的数据库备份恢复到内网的机器，用来分析数据，但备份中的某些很大的表对我们的数据分析作用不大，却要占用大量的导入时间（譬如session表，message表），我们可以用简单的sed命令跳过对这些表的插入操作。

{% highlight shell %}

sed '/INSERT INTO `sessions`/d' db.sql > db_new.sql

{% endhighlight %}

### 找出最近修改、创建的文件

以下命令可以用来简单的检查最近目录下哪些文件被修改过。

{% highlight shell %}

find . -newermt "2013-01-01 00:00:00" ! -newermt "2013-01-02 00:00:00"

find . -mtime -10 -mtime +4

# find has + and - operartor, Also *time : mtime, atime and ctime : 
# atime == Acccess Time 
# mtime == Modified Time 
# ctime == Create Time

{% endhighlight %}

### 相关文章

* [自动压缩旧日志文件的shell脚本](http://yplam.com/linux/2016/10/12/shell-gzip-log-file.html)
* [Crontab 控制脚本运行时间段](http://yplam.com/linux/2016/08/12/linux-crontab-run-time.html)
