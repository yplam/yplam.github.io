---
title:  "Crontab 控制脚本运行时间段"
date:   2016-08-12
categories: Linux
---

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


