---
title:  "一步步搭建 Edison ROS Indigo 环境"
categories: ROS
---

最近在琢磨能不能用 CNN 来训练一个模型，用于控制一个模型小车，实现简单的循迹或者所谓的自动驾驶。于是拿出布灰已久的 Edison + 两轮平衡车 + 罗技摄像头，计划将这几样组装起来，上面运行 ROS 作为我的“机器人”平台，然后在笔记本上运行 master 做运算与控制。本笔记记录 ROS 在 Edison 上安装与运行 usb_cam 获取图像传输到主机的过程。

主要还是按照 [Installing ROS Indigo on Intel Edison](http://wiki.ros.org/wiki/edison), [Installing ROS Indigo on Raspberry Pi](http://wiki.ros.org/ROSberryPi/Installing%20ROS%20Indigo%20on%20Raspberry%20Pi) 这两个文档进行安装，但过程中会遇到不少问题，并且由于 Edison 用户可能比较少，大多都需要找相关资料来解决。

本笔记假设读者已经拥有基础 Linux 操作技能，某些基础操作不会细说。

### 安装 ubilinux

参考下面文档：

https://learn.sparkfun.com/tutorials/loading-debian-ubilinux-on-the-edison

### 将 ubilinux 安装到 sd 卡启动

因为 Edison 自带的 emmc 只有 4G，安装各种包之后空间只能勉强够用，因此建议使用 8G 以上的 sd 卡安装系统，安装在sd卡上还有一个好处就是可以简单的将系统备份下来。可参考下面文档：

https://github.com/sarweshkumar47/Boot-Intel-Edison-from-SDCard-with-Debian-Ubilinux


### 系统初始化

配置 wifi：

```
wpa_passphrase wifiid wifi_password
```

然后修改 /etc/network/interfaces，禁用 usb 网卡，使用 wlan0，将 wpa-ssid 跟 wpa-psk 配置上。

初始化的配置，顺便把 locale 配置一下

```
apt-get update
dpkg-reconfigure locale
```



### 安装 boost 1.5.3 库

Debian Wheezy 自带的 boost 库版本为 1.4.9，如果按此版本编译安装 ROS Indigo 的话会报以下错误：

```
error: macro "BOOST_SCOPE_EXIT" passed 2 arguments, but takes just 1

```

查看相关文档得知，Indigo 对应的 boost 版本为 1.5.3，可以按 [Getting Started on Unix Variants](http://www.boost.org/doc/libs/1_53_0/more/getting_started/unix-variants.html) 编译安装，也可以下载我编译好的 deb 包进行安装（编译一次需要 N 个小时）。当然，如果你有兴趣的话也可以尝试编译自己的deb包，参考 https://github.com/ulikoehler/deb-buildscripts/blob/master/deb-boost.sh

下载 deb 包： [boost-all_1.53.0-1_i386.deb]({{ site.url }}/assets/2017/boost-all_1.53.0-1_i386.deb) [boost-all-dev_1.53.0-1_i386.deb]({{ site.url }}/assets/2017/boost-all-dev_1.53.0-1_i386.deb)

安装：

```
apt-get install libicu48
dpkg -i boost-all_1.53.0-1_i386.deb boost-all-dev_1.53.0-1_i386.deb

```

### 安装 opencv 库

编译安装：

下载opencv 2.4.9 代码

环境准备

```
apt-get install build-essential
apt-get install cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev ffmpeg
apt-get install python-dev python-numpy


```

根据这里对代码进行修改 https://github.com/NextThingCo/buildroot/blob/master/package/opencv/0002-superres-Fix-return-type-value-VideoFrameSource_GPU.patch

263 行

```

-    return new VideoFrameSource(fileName);
+    return new VideoFrameSource_GPU(fileName);

```

在 CMakeLists.txt 中 set(OPENCV_VCSVERSION "2.4.9")


```
mkdir release
cd release
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local ..

编辑 CMakeCache.txt CPACK_BINARY_DEB:BOOL=ON

make
make install
```

上面这步可能需要运行 N 个小时

```
make package

```

可能会出错

```
CPackDeb: dpkg-shlibdeps: dpkg-shlibdeps: need at least one executable
```

按照这里的 patch 对 cmake 进行修正：https://public.kitware.com/Bug/view.php?id=13488

附预编译的 deb 包： [OpenCV-2.4.9-i686-libs.deb]({{ site.url }}/assets/2017/OpenCV-2.4.9-i686-libs.deb) [OpenCV-2.4.9-i686-dev.deb]({{ site.url }}/assets/2017/OpenCV-2.4.9-i686-dev.deb) [OpenCV-2.4.9-i686-python.deb]({{ site.url }}/assets/2017/OpenCV-2.4.9-i686-python.deb)

## 编辑安装 ROS

可以参考 [Installing ROS Indigo on Intel Edison](http://wiki.ros.org/wiki/edison), [Installing ROS Indigo on Raspberry Pi](http://wiki.ros.org/ROSberryPi/Installing%20ROS%20Indigo%20on%20Raspberry%20Pi) 进行安装，但有两点区别：

**1.2 console_bridge:**

在运行下面命令后

```
sudo checkinstall make install

```

输入 2 并且将包名改为 'libconsole_bridge' .


**3.2.2 Resolving Dependencies with rosdep**

运行 rosdep install 的时候去掉 -y 参数，并且跳过 apt-get install libboost-all-dev.

**error while loading shared libraries: libconsole_bridge.so.0.3: cannot open shared object file: No such file or directory**

如果你遇到这个错误，可以通过下面方式处理

```
cd /opt/ros/indigo/lib
sudo ln -s /usr/local/lib/i386-linux-gnu/libconsole_bridge.so.0.3 libconsole_bridge.so.0.3

```

## 编译 usb_cam

usb_cam 的编译跟前面 ros_comm 类似：

```
mkdir catkin_ws
cd catkin_ws
mkdir src
rosinstall_generator usb_cam image_transport_plugins --rosdistro indigo --deps --wet-only --exclude roslisp --tar > indigo-usb_cam.rosinstall
wstool init src indigo-usb_cam.rosinstall
rosdep install --from-paths src --ignore-src --rosdistro indigo -r --os=debian:wheezy
#跳过opencv boost相关的包
source /opt/ros/indigo/setup.bash
catkin_make
```

你也可以在之前的工作区上操作，因为新工作区会导致某些包又要重新编译，但好处就是以后 make 跑的时间会短一点。

## 运行 usb_cam 将 Edison 图像传输到电脑

开始运行之前请确认主机跟从机都可以相互ping通，并且可以ping通自己。

现在主机运行：

```
roscore
```

主机上开多一个终端，运行：

```
rosrun rviz rviz
```

Edison上编辑 launch 文件，运行

```
source devel/setup.bash
roscd usb_cam
vim launch/usb_cam.launch
roslaunch usb_cam.launch

```

usb_cam.launch 的内容，假设你主机的名字为 lam

```
<launch>
  <env name="ROS_MASTER_URI" value="http://lam.local:11311"/> 
  <node name="usb_cam" pkg="usb_cam" type="usb_cam_node" output="screen" >
    <param name="video_device" value="/dev/video0" />
    <param name="image_width" value="640" />
    <param name="image_height" value="480" />
    <param name="pixel_format" value="yuyv" />
    <param name="camera_frame_id" value="usb_cam" />
    <param name="io_method" value="mmap"/>
  </node>
</launch>

```

在 rviz 窗口上添加一个 Image，topic 为 /usb_cam/image_raw ，使用 compressed 传输。


Done.
