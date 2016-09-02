---
title:  "Linux 初始化 SSH 会话慢的解决方法"
date:   2016-09-01 10:51:00 +0800
categories: Linux
---

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
