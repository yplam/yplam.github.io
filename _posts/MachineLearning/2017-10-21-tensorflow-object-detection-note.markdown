---
title:  "Tensorflow Object Detection API 入门笔记 - 基于 Google Cloud 与 阿里云"
categories: MachineLearning
published: true
---

跟风关注机器学习已有一段时间，最近需要做一个图像识别的项目，刚好 Google 开源了 [Tensorflow Object Detection API](https://github.com/tensorflow/models/tree/master/research/object_detection) ，
于是在此基础上做了一次尝试。本笔记从一名程序员的角度记录一次基于 Tensorflow Object Detection API 的图像物体识别项目过程，项目使用 Google Cloud Machine Learning 进行模型训练，
使用阿里云 GPU 服务器进行模型测试评估。

**目的：**在一批图片中找出包含物体A的图片，并且标出图片中各物体A的位置。

**环境：**Ubuntu 16.04，python 2.7，virtualenv

**过程：**

1. 安装 Tensorflow 与 Tensorflow models
2. 获取与标注图片
3. 生成训练集与测试集
4. 配置、提交 Google Cloud 训练
5. 使用阿里云 GPU 服务器进行测试

## 安装

CPU 环境下的 Tensorflow 安装非常简单，可以参考[官网安装文档](https://www.tensorflow.org/install/install_linux)，其中 python 版本建议选择 2.7，因为现时（2017年10月） Google Cloud Machine Learning 的命令行对 python3 支持有问题。

GPU 下的安装稍微复杂，主要是需要解决 NVIDIA 显卡驱动问题，可以参考之前的文章 [http://yplam.com/machinelearning/2016/12/12/aws-cuda-tensorflow.html](http://yplam.com/machinelearning/2016/12/12/aws-cuda-tensorflow.html)。需要注意的是1.3官方编译版本的Tensorflow使用的是8.0版本的cuda以及6.0版本的cudnn，可以在[发布说明页面](https://github.com/tensorflow/tensorflow/releases)搜"CUDA"找到相关说明。

如果你需要在阿里云的GPU服务器上使用，可以使用[这个Ansible脚本](https://github.com/yplam/AliyunTensorflowGPUAnsible)进行快速初始化。

简要步骤如下：(详细参考[https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/installation.md](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/installation.md))

```
# For CPU
pip install tensorflow
# For GPU
pip install tensorflow-gpu

apt-get install protobuf-compiler

pip install pillow
pip install lxml
pip install jupyter
pip install matplotlib
```

编译Protobuf

```
cd tensorflow/models/research/
protoc object_detection/protos/*.proto --python_out=.
```

编辑~/.bashrc，将 Object Detection 库加入python（注：请根据你的实际路径设置）

```
# From tensorflow/models/research/
export PYTHONPATH=$PYTHONPATH:tensorflow/models/research:tensorflow/models/research/slim
```

安装完成后可以跑一下下面的 jupyter notebook 测试 research/object_detection/object_detection_tutorial.ipynb

## 数据获取与标注

在进行下面步骤前可以先看看官方的这个文档[Quick Start: Distributed Training on the Oxford-IIIT Pets Dataset on Google Cloud](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/running_pets.md)，下面的内容均是基于官方文档并且根据本项目情况进行修改。

所谓数据就是用于训练与测试的预标注图片。图片可以是预先准备好的，或者是通过 Scrapy 等爬虫工具获取。图片可能大小不一，下载下来后可以通过下面命令缩小到1000px宽度：

```
convert '*.jpg[1000x]' -set filename:base "%[base]" "1000/%[filename:base].jpg"
```

需要注意的是过小的图片可能无法在某些模型中使用。

然后就到了整个过程中最耗时最无聊的步骤：数据标注（所谓人工智能真的有一大堆人工...）。

在此使用 [labelimg](https://github.com/tzutalin/labelImg) 进行图片标注，安装与使用方法请参考github项目页面。分享一个技巧就是熟记快捷键，如果有多个类别的话可以设定默认label，并且每次只标注一个类别。

图片标注后会在目标目录生成一个同名的 xml 文件。

## 生成训练集与测试集

Tensorflow Object Detection 使用的是 TFRecord 格式的数据，所以需要将上面标注好的图片与xml文件根据模型的数据格式需要打包到一个 TFRecord 文件中。

首先，我们需要一个 label.pbtxt 文件，标识分类 id 以及 label 的对应关系，如：

```
item {
 id: 1
 name: 'toy'
}

```

然后参考 research/object_detection/create_pet_tf_record.py ，编写自己的 python 脚本生成你的 训练集与测试集 record 文件。

## 配置与提交到 Google Cloud 进行模型训练

Tensorflow models 项目页面提供多中模型的[配置范例](https://github.com/tensorflow/models/tree/master/research/object_detection/samples/configs)，在此我们使用预训练的 faster_rcnn_resnet101 模型。

由于需要使用 Google Cloud Machine Learning 服务进行模型训练，所以需要先安装 Google Cloud 相关的客户端，开通 Google Cloud 账号，并且启用 Machine Learning 与 Cloud Storage。可参考下面链接：

* [Create a GCP project.](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
* [Install the Google Cloud SDK.](https://cloud.google.com/sdk/downloads)
* [Enable the ML Engine APIs.](https://console.cloud.google.com/flows/enableapi?apiid=ml.googleapis.com,compute_component&_ga=1.73374291.1570145678.1496689256)
* [Set up a Google Cloud Storage (GCS) bucket.](https://cloud.google.com/storage/docs/creating-buckets)

如果你使用的是ubuntu系统的话这个过程应该还算简单。当然，这还需要你具备“科学上网”技能，以及proxychains等工具的使用。

将我们前面生成的record文件以及label.pbtxt文件拷贝到Google Cloud Storage，如（请根据实际情况修改）：

```
# From tensorflow/models/research/
gsutil cp pet_train.record gs://${YOUR_GCS_BUCKET}/data/pet_train.record
gsutil cp pet_val.record gs://${YOUR_GCS_BUCKET}/data/pet_val.record
gsutil cp object_detection/data/label.pbtxt gs://${YOUR_GCS_BUCKET}/data/label.pbtxt
```

下载 COCO-pretrained Faster R-CNN with Resnet-101 模型，解压到 Google Cloud，解压的文件本地也要留一份，后面会用到。

```
wget http://storage.googleapis.com/download.tensorflow.org/models/object_detection/faster_rcnn_resnet101_coco_11_06_2017.tar.gz
tar -xvf faster_rcnn_resnet101_coco_11_06_2017.tar.gz
gsutil cp faster_rcnn_resnet101_coco_11_06_2017/model.ckpt.* gs://${YOUR_GCS_BUCKET}/data/
```

参考 faster_rcnn_resnet101_pets.config ，编写我们的config文件，主要是修改 num_classes，fine_tune_checkpoint，input_path，label_map_path，可以先使用本地路径，而不是官方教程中的 "gs://***" 路径，方便本地调试好再发上去正式训练。因为发到 Google Cloud 训练会有个初始化等待过程，并且由于调用的GPU数量较多，如果因为配置文件不符而导致需要不断修改调试，在这过程中可能会浪费比较多的时间跟费用。

现在，你本地会有以下文件(名字可能不一样)：

```
    - faster_rcnn_resnet101_pets.config
    - model.ckpt.index
    - model.ckpt.meta
    - model.ckpt.data-00000-of-00001
    - label.pbtxt
    - pet_train.record
    - pet_val.record
```

先在本地测试一下是否能正常训练，开始前尽量把不必要的其他应用关掉，因为将耗费 N 多 cpu 与内存资源。

```
python object_detection/train.py --train_dir=tmp --pipeline_config_path=${YOUR_CONFIG_FILE}.config

```

如果能正常进入训练进程，则说明各种配置正常，那么可以将 faster_rcnn_resnet101_pets.config 中的文件路径设为 gs:// 上面路径。然后提交训练（提交前先将本地faster_rcnn_resnet101_pets.config拷贝到gs://${YOUR_GCS_BUCKET}/data）

```

# From tensorflow/models/research/
python setup.py sdist
(cd slim && python setup.py sdist)

```

```
# From tensorflow/models/research/
gcloud ml-engine jobs submit training `whoami`_object_detection_`date +%s` \
    --job-dir=gs://${YOUR_GCS_BUCKET}/train \
    --packages dist/object_detection-0.1.tar.gz,slim/dist/slim-0.1.tar.gz \
    --module-name object_detection.train \
    --region us-central1 \
    --config object_detection/samples/cloud/cloud.yml \
    -- \
    --train_dir=gs://${YOUR_GCS_BUCKET}/train \
    --pipeline_config_path=gs://${YOUR_GCS_BUCKET}/data/faster_rcnn_resnet101_pets.config
```

然后可以登录 Google Cloud Console 查看训练进度，需要注意的是费用大概是 $10 一小时，所以如果看到loss降低到一个合适的程度时可以提前停止训练。

训练后的模型会存储在配置的 --train_dir 中，包含类似下面三个文件：

```
model.ckpt-${CHECKPOINT_NUMBER}.data-00000-of-00001
model.ckpt-${CHECKPOINT_NUMBER}.index
model.ckpt-${CHECKPOINT_NUMBER}.meta
```

下载下来后可以通过下面命令导出：

```
python object_detection/export_inference_graph.py \
    --input_type image_tensor \
    --pipeline_config_path object_detection/samples/configs/faster_rcnn_resnet101_pets.config \
    --trained_checkpoint_prefix model.ckpt-${CHECKPOINT_NUMBER} \
    --output_directory output_inference_graph.pb
    
```

output_inference_graph.pb 文件就是导出后的模型文件，可以留作后续使用。


## 在阿里云上运行训练后的模型

Resnet 101 模型在本地 i5 台式机上跑预测一次时间大概是30秒，而在阿里云最低配的 M40 GPU AWS 上运行一次只需要不到1秒，所以对于大批量的预测任务，配置一台阿里云的GPU服务器来跑是一个不错的选择。现时阿里云 M40 服务器价格为 ￥10 一小时，相对于动辄过万的GPU台式机，还算是个相对便宜的选择。

使用[这个Ansible脚本](https://github.com/yplam/AliyunTensorflowGPUAnsible)进行快速初始化Tensorflow环境，本脚本在 阿里云M40 GPU AWS、UBUNTU 16.04、cuda 8、tensorflow 1.3 上测试通过。


参考资料：

* [Tensorflow Object Detection API](https://github.com/tensorflow/models/tree/master/research/object_detection)
* [Building a Toy Detector with Tensorflow Object Detection API](https://medium.com/towards-data-science/building-a-toy-detector-with-tensorflow-object-detection-api-63c0fdf2ac95)
* [A Brief History of CNNs in Image Segmentation: From R-CNN to Mask R-CNN](https://blog.athelas.com/a-brief-history-of-cnns-in-image-segmentation-from-r-cnn-to-mask-r-cnn-34ea83205de4)
