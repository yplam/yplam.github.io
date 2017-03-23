---
title:  "Install ROS Indigo on Edison Ubilinux"
categories: ROS
---

This tutorial shows you how to install ROS Indigo on Edison Ubilinux from source.

You can mainly follow these instructions [Installing ROS Indigo on Intel Edison](http://wiki.ros.org/wiki/edison), [Installing ROS Indigo on Raspberry Pi](http://wiki.ros.org/ROSberryPi/Installing%20ROS%20Indigo%20on%20Raspberry%20Pi) except the following differences:

** compile and install boost 1.5.3 **

Follow this instruction [Getting Started on Unix Variants](http://www.boost.org/doc/libs/1_53_0/more/getting_started/unix-variants.html) to install boost 1.5.3 because the default boost 1.4.9 on wheezy does not works with ROS Indigo.

Without installing boost 1.5.3, you may get the error message like this:

```
error: macro "BOOST_SCOPE_EXIT" passed 2 arguments, but takes just 1

```

** 1.2 console_bridge: **

In :

```
sudo checkinstall make install

```

Enter 2 and rename the package to 'libconsole_bridge' .

** 3.2.2 Resolving Dependencies with rosdep **

Run rosdep install without the -y argument and skip apt-get install boost* lib.

**error while loading shared libraries: libconsole_bridge.so.0.3: cannot open shared object file: No such file or directory**

After the whole install, if you got this error while starting roscore, you can fix it like this:

```
cd /opt/ros/indigo/lib
sudo ln -s /usr/local/lib/i386-linux-gnu/libconsole_bridge.so.0.3 libconsole_bridge.so.0.3

```
