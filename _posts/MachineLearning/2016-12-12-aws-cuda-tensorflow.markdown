---
title:  "AWS TensorFlow 安装与测试"
categories: MachineLearning
---

最近迷上了机器学习，花了不少时间学习相关的基础内容；虽然数学跟不上，也不妨碍跟风玩玩各位大牛的开源成果。
本文记录了购买AWS GPU服务器，安装cuda、 TensorFlow，运行 Neural Style 测试的过程。


### 购买AWS EC2 p2.xlarge

购买p2.xlarge实例（1 NVIDIA K80 GPU， 4核CPU，61G内存，$0.9/小时），空间需要10G以上，建议20G，因为nvidia与cuda包的安装都需要不少的空间（按下面的安装方法，使用的空间接近10G）。
选择 aws 提供的 ubuntu 16.04，跟需求配置安装。

安装后建议设置一下locale，不然下面的步骤中可能会报错。

```
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo dpkg-reconfigure locales
```

### 安装Nvidia显卡驱动

首先，检查一下服务器中是否真的存在Nvidia显卡：

```
$ lspci | grep -i nvidia
```

直接使用ubuntu官方提供的Nvidia显卡驱动，目前版本是367：

```
apt-cache search nvidia
sudo apt-get install nvidia-367 nvidia-modprobe

```

检测驱动是否正常安装：

```
nvidia-smi
```

正常情况下会输出：

```

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 367.57                 Driver Version: 367.57                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 0000:00:1E.0     Off |                    0 |
| N/A   29C    P0    74W / 149W |      0MiB / 11439MiB |     99%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+


```


### 安装cuda cuDNN

到[官网](https://developer.nvidia.com/cuda-downloads)下载最新版本的cuda，以及对应的cudnn，安装：

```
wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_8.0.44-1_amd64.deb

sudo dpkg -i cuda-repo-ubuntu1604_8.0.44-1_amd64.deb
sudo apt-get update
sudo apt-get install cuda
```

下载与安装cuDNN（需要到官方注册登录后才能得到下载地址）

```
tar xvzf cudnn-8.0-linux-x64-v5.1.tgz
sudo cp -P cuda/include/cudnn.h /usr/local/cuda/include
sudo cp -P cuda/lib64/libcudnn* /usr/local/cuda/lib64
sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*

```

### 安装TensorFlow

```
sudo apt-get install python-pip python-dev
sudo pip install --upgrade pip

sudo pip install tensorflow-gpu

```

### 测试Neural Style

```
sudo apt-get install libatlas-base-dev
sudo pip install numpy scipy pillow

git clone https://github.com/anishathalye/neural-style.git
cd neural-style
wget http://www.vlfeat.org/matconvnet/models/beta16/imagenet-vgg-verydeep-19.mat

python neural_style.py --content examples/1-content.jpg --styles examples/1-style.jpg --output 1.jpg

```

完成。

跑一次 Neural Style 大概需要7分钟。

测试后记得讲实例停掉，然后制作image，留作下次测试用，讲运行中的实例删除，毕竟$0.9/小时也不便宜（image保存在EBS中，也会消耗小量的费用）。

玩了一天5个小时，消耗4.5刀，虽然不算便宜，但相比动辄过万的深度学习工作站，平时玩玩的话还算便宜。

作品：

![input]({{ site.url }}/assets/2016/in.jpeg "input") ![output]({{ site.url }}/assets/2016/out.jpg "output")



