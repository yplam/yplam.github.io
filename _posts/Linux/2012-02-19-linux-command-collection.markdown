---
title:  "Linux 下的一些技巧"
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

### 自动压缩旧日志文件的shell脚本

logrotate 是 Linux 自带的日志分割压缩工具，但它有个比较让人不爽的缺点就是无法精确的按时间点分割（譬如日志严格的按天）。因此考虑App日志直接按 ×××-Y-m-d.log的方式来按天记录，然后用crontab定时压缩。

下面是自动gzip压缩多个目录下log文件，并且自动跳过当天文件的shell脚本：


{% highlight shell %}

#!/bin/sh

FOLDERS=(/data/logs/view/ /data/logs/search/)
NOW="$(date +"%Y-%m-%d")"
for FOLDER in ${FOLDERS[@]}
do
	echo "Zipping $FOLDER"
	for FILE in ${FOLDER}*.log
	do
		if [[ ! $FILE =~ $NOW ]]; then
			echo "$FILE"
			gzip -9 $FILE
		fi
	done
done


{% endhighlight %}

### Linux 初始化 SSH 会话慢的解决方法

使用ssh登录Linux服务器，就算是在内网环境中，也需要等待比较长的时间才弹出密码输入界面。

检查：

ssh -vvv 加上 v参数，通过分析ssh打印的debug信息，即可找到时间消耗的环节。

{% highlight shell %}

$ ssh -vvv root@*****
OpenSSH_6.6.1, OpenSSL 1.0.1f 6 Jan 2014
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 19: Applying options for *
debug2: ssh_connect: needpriv 0
debug1: Connecting to @***** port 22.
debug1: Connection established.
debug3: Incorrect RSA1 identifier
debug1: Enabling compatibility mode for protocol 2.0
debug1: Local version string SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.8
debug1: Remote protocol version 2.0, remote software version OpenSSH_5.3
debug1: match: OpenSSH_5.3 pat OpenSSH_5* compat 0x0c000000
debug2: fd 3 setting O_NONBLOCK

{% endhighlight %}

可能原因：

服务端 /etc/ssh/sshd_config，修改 "UseDNS no"

客户端  /etc/ssh/ssh_config，修改 "GSSAPIAuthentication no"

### Crontab 控制脚本运行时间段

Cronjob 通常用来在指定时间点运行某个命令（脚本），而有时我们不但需要定时启动脚本，还需要定时关闭，譬如需要在空闲时段备份服务器文件到新机器。

我们可以使用两个cronjob，一个定时启动，一个定时关闭，或者使用类似下面的脚本，启动后后台运行一个进程，通过获取时间来定时关闭：


{% highlight shell %}

#!/bin/bash

rsync ... &
pid=$!

while /bin/true; do
  if [ $(date +%H) -ge 8 ]; then
    kill -TERM $pid
    exit 0
  else
    sleep 60
  fi
done

{% endhighlight %}

### rsync 只同步匹配的目录文件

通常我们的图片、文件会根据某中规则（譬如2016-09-09）的格式存储在文件系统中，有时我们需要将适合某个

{% highlight shell %}

rsync -aPv --include='**/2016-09-09/*' --include='*/' --exclude='*' rsync://...

{% endhighlight %}

