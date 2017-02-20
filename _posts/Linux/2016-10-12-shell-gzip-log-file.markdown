---
title:  "自动压缩旧日志文件的shell脚本"
date:   2016-10-12
categories: Linux
---

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


