---
title:  "Debian 8 服务器运行多Redis实例"
categories: Linux Redis
---

Redis 由于功能强大、性能优越、支持持久化等优点，在很多应用上已经替代Memcached，成为首选的缓存系统，还可以作为简单的db使用。

正因为如此，有时我们需要将redis同时作为下面几种功能使用：

- cache，不需要持久化
- session，只需要简单的持久化，隔个十分钟存一下就好，丢失一些也不会造成太大问题
- db，不能丢失数据，需要比较积极的持久化策略，日志记录等

虽然你可以按db的方式来统一对待所有的需求，将它们放在同一个实例中，使用不同的database区分，但是将它们分到不同实例的方式可能会更好。


**配置文件**

配置文件位与 /etc/redis 目录，默认有 redis.conf

我们可以将redis.conf重命名为common.conf，然后用include的方式来定制每一个redis实例的配置

如 redis-session.conf

```
include /etc/redis/common.conf
pidfile /var/run/redis/redis-session.pid
port 6380

save 900 1
save 60 10000
dbfilename dump-session.rdb

```

**Service配置**

如果你想将你自定义的redis实例变成系统的service，使用debian自带的service命令来启动重启，那么你需要配置service脚本

复制 /lib/systemd/system/redis-server.service 到 /lib/systemd/system/redis-session.service 

然后根据需求修改里面的配置，主要是配置文件，pid路径等。

你可以通过 service redis-session start 或者 status 等命令来测试，如果启动失败，请通过 /var/log/syslog 文件来查看问题

**INIT脚本**

如果你想你的redis实例可以开机自动运行，那么就需要配置init脚本

复制 /etc/init.d/redis-server 到 /etc/init.d/redis-session

根据需求修改里面的配置，主要也是命名、配置等信息，然后通过

```
update-rc.d redis-session defaults
```

命令来设置开机自动运行
