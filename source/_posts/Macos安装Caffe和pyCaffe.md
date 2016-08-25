---
title: Macos安装Caffe和pyCaffe
date: 2016-08-19 13:21:35
tags: [Caffe, MacOS]
---
## 参考阅读

其实主要是翻译的官方文档，官方文档主要是比较全面，各种情况都要考虑到，所以内容比较分散，而很多细节容易被忽略掉。但是如果具体我自己的安装需求，情况比较单一，所以相比之下更加清晰一点。

[Macos依赖安装](http://caffe.berkeleyvision.org/install_osx.html)
[源码编译](http://caffe.berkeleyvision.org/installation.html#compilation)

## 安装依赖包

官方的代码为：

~~~shell
brew install -vd snappy leveldb gflags glog szip lmdb
# need the homebrew science source for OpenCV and hdf5
brew tap homebrew/science
brew install hdf5 opencv
~~~

依次安装就好，每个包的作用如下：

### snappy
用于压缩和解压文件

### leveldb & lmdb
将各种原始数据通过Key-Value键值对的方式存储在内存中，方便Caffe的DataLayer获取这些数据。 lmdb是比较新的数据管理库，leveldb是比较老的版本，但是为了兼容，仍然保留了这个库。

### gflags
解析命令行参数。

### glog 
日志记录。

### BLAS
我不记得有安装这个，可能之前其他项目有装过，这个是主要的矩阵、向量计算的库。根据文档，应该默认使用的`ATLAS`库。

### HDF5
统一的数据存储格式。

### OpenCV
Caffe仅仅使用了图片读写、缩放等基本操作。

## 编译源码

首先，使用默认的配置文件作为我们编译过程的基础配置：

~~~shell
cp Makefile.config.example Makefile.config
~~~

由于我的mbp没有可用的GPU，所以将配置文件里的` CPU_ONLY := 1`项去掉注释。

然后依次执行

~~~shell
make all -j8
make test
make runtest
~~~

其中，`make all -j8`中的数字代表并行编译的线程，依据机器的真实情况修改，可以极大地提高编译的速度。

最后编译pyCaffe即可

~~~shell
make pycaffe 
~~~

## 小结

这一次安装的异常顺利，远远没有年前那次安装得痛苦。期望之后通过使用`Caffe`学到三个方面的内容，第一当然是深度学习模型的原理、训练、使用等；第二是通过阅读源码学习`C++`的程序结构和各类库的使用；第三是`Python`和`C++`的接口实现。