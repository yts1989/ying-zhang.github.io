
title: CentOS 7安装TensorFlow GPU深度学习环境
category: [cloud]
tags: 
date: 2017-12-04
---

在CentOS 7上安装Nvidia GTX 1080 Ti显卡的驱动，以及TensorFlow GPU等深度学习开发环境。
就这么突然，跟深度学习扯上边了；-)

-----
<!--more-->

上周老板突然说要给机房的Dell服务器分别装两个显卡，让我去看一下，然后把支持GPU的深度学习开发环境搭起来。装显卡是供应商的一个小哥动手的，基本顺利，遇到的小问题是电源供电不足，需要改一下iDrac中的电源设置，将服务器的两路电源互为备用模式改为两路同时供电，这样功率才够跑两个显卡。

网上一搜，就有不少CentOS上搭环境的文章了，但 1）相关开源项目发展太快，2）不同需求的用户可以有针对性的简化配置过程，所以我把集群上实测过的步骤记录下来。因为自己完全是门外汉，所以还没有涉及具体的深度学习知识。

# 简介
网上相关文章的步骤大多是先安装驱动，再安装CUDA，还需要安装C++编译器（g++或msvc），再安装cuDNN库，最后通过`pip`或`conda`再安装tensorflow。[TensorFlow官网上的安装说明](https://www.tensorflow.org/install/install_linux)以及 [Nvidia官网上的安装说明](https://www.nvidia.com/en-us/data-center/gpu-accelerated-applications/tensorflow/)亦是如此。

CUDA（Compute Unified Device Architecture，统一计算架构）是针对GPU计算加速的开发工具包，就像Windows SDK，或者JDK一样，一些深度学习库（比如TensorFlow）的底层是C++调用的CUDA库，它们提供给深度学习开发者的多是 Python 包装过的接口。一般的开发者直接用这些Python库就可以设计出多种多样的深度学习模型，不再需要跟CUDA打交道。

如果**不需要从源码编译TensorFlow**，就没必要安装NVIDIA官网上的那个一个多GB的CUDA包和cuDNN库。直接通过<del>`pip`或</del>`conda`安装的`tensorflow-gpu`库就自带了对应版本的cuda动态链接库，包括 **libnvrtc-builtins.so，libnvrtc.so，libnvToolsExt.so，libnvvm.so，libcudart.so，libcublas.so，libcudnn.so，libcurand.so，libcufft.so，libcusolver.so，libcusparse.so** 等，还有mkl库（Linux的是`.so`文件，Windows的是`.dll`文件）。

> 注意，见下面关于`pip`和`conda`的小节。

最近（2017-12-4）Nvidia官网上的CUDA版本已经是9.0，而TensorFlow 1.4 使用的是cuda 8.0，cuDNN则是6.0，python又有2.7、3.5、3.6版。各种版本组合起来还有点麻烦呢。我们先从显卡驱动开始。

# 安装显卡驱动
先看看显卡硬件是不是安装好了，执行`lspci | grep NVIDIA`，可见已经安装了两个GeForce GTX 1080 Ti显卡：
```
[root@n170 ~]# lspci | grep NVIDIA
03:00.0 VGA compatible controller: NVIDIA Corporation GP102 [GeForce GTX 1080 Ti] (rev a1)
03:00.1 Audio device: NVIDIA Corporation GP102 HDMI Audio Controller (rev a1)
82:00.0 VGA compatible controller: NVIDIA Corporation GP102 [GeForce GTX 1080 Ti] (rev a1)
82:00.1 Audio device: NVIDIA Corporation GP102 HDMI Audio Controller (rev a1)
```

然后安装驱动：
```
# 参考：https://www.dedoimedo.com/computers/centos-7-nvidia-second.html
# 及 https://www.youtube.com/watch?v=C9Yf71qh0i4

sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
sudo yum install   nvidia-detect  # 这个命令的输入就是 kmod-nvidia，所以不安装也可以。。。
sudo yum install $(nvida-detect)
sudo yum install   kmod-nvidia
sudo reboot # 重启是必须的
```

> Ubuntu的命令是
```
# 参考：https://www.nvidia.com/en-us/data-center/gpu-accelerated-applications/tensorflow/

sudo add-apt-repository ppa:graphics-drivers/ppa 
sudo apt update
sudo apt install nvidia- # 敲到 nvidia- 后按一下Tab键，稍等一会，会列出补全项，显示目前最新的是387，
# 就是说完整的命令是

sudo apt install nvidia-387

# 注意，不要选择 378 版，否则会造成无限重试登录
# 安装后也要重启系统
```

重启后查看驱动是否安装正确，执行`nvidia-smi`（还可以执行`watch -n 1 nvidia-smi`持续监控）：
```
[root@n170 ~]# nvidia-smi
Mon Dec  4 16:03:57 2017       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.98                 Driver Version: 384.98                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:03:00.0 Off |                  N/A |
|  0%   29C    P8     8W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GeForce GTX 108...  Off  | 00000000:82:00.0 Off |                  N/A |
|  0%   30C    P8     9W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

还可以执行`cat /proc/driver/nvidia/version`
```
[root@n170 ~]# cat /proc/driver/nvidia/version 
NVRM version: NVIDIA UNIX x86_64 Kernel Module  384.98  Thu Oct 26 15:16:01 PDT 2017
GCC version:  gcc version 4.8.5 20150623 (Red Hat 4.8.5-16) (GCC)
```

`gpustat`是一个输出格式比较简单的工具，通过`pip install gpustat`安装后，输出格式如下（其中n170是机器名）：
```
[root@n170 ~]# gpustat
n170  Mon Dec  4 16:07:10 2017
[0] GeForce GTX 1080 Ti | 28'C,   0 % |     0 / 11172 MB |
[1] GeForce GTX 1080 Ti | 31'C,   0 % |     0 / 11172 MB |
```

# 安装 Anaconda 和 Python 3.6
这里选择的Python版本是3.6，但不是从Python官网或yum安装的，而是Anaconda集成环境内置的版本，这个集成环境还有`conda`包管理器，`jupyter notebook`和`numpy`，`pandas`等一些常用的包。

Anaconda官网下载页是[https://www.anaconda.com/download/] ，不过我们从清华的镜像站下载，这样下载速度快一点[https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-5.0.1-Linux-x86_64.sh] 。虽然这个安装文件后缀是`.sh`，但实际的二进制安装文件都打包在里面了，有525MB。

安装过程需要用到`bzip2`，先安装一下`sudo yum install -y bzip2`
执行 `bash Anaconda3-5.0.1-Linux-x86_64.sh` 开始安装，敲回车显示 license agreement ，敲几次空格翻到底，然后输入`yes`接受协议，再敲回车，安装到默认的路径`$HOME/anaconda3`，如果这个路径已经存在，就会安装失败，需要删掉或另选路径。
安装脚本还会在`.bashrc`的`PATH`环境变量加上安装路径。安装结束后，执行`source .bashrc`，更新`PATH`环境变量，这时系统的`python`命令已经变成Anaconda安装的Python 3.6了（因为安装程序把`$HOME/anaconda3/bin`加在了`PATH`最前面）。

> 更改`conda`源：执行
`conda config --add channels 'https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/'`
`conda config --set show_channel_urls yes`
这两个命令其实是把配置项写到了`~/.condarc`文件，还可以在这里设置http代理：
```
channels:
  - defaults
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
show_channel_urls: true

proxy_servers:
    http:  http://127.0.0.1:1080
    https: http://127.0.0.1:1080

ssl_verify: False
```

# 安装 TensorFlow

执行 `conda install tensorflow-gpu`，注意，安装的版本是 `1.3.0-py36cuda8.0cudnn6.0_1` ，不是最新的`1.4.0`版，不过好处是开箱即用，就这句命令就搞定了。cuda相关的动态库都已经安装在了`$HOME/anaconda3/lib`。

执行一下官网的测试例子：
```
[root@n170 ~]# python
Python 3.6.2 |Anaconda custom (64-bit)| (default, Sep 22 2017, 02:03:08) 
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> hello = tf.constant('Hello, TensorFlow!')
>>> sess = tf.Session()
>>> print(sess.run(hello))
b'Hello, TensorFlow!'
>>> with tf.Session():
...     a=tf.constant([1.0, 1.0, 1.0, 1.0])
...     b=tf.constant(2.0, shape=[4])
...     out=tf.add(a,b)
...     print("result:",out.eval())
... 
result: [ 3.  3.  3.  3.]
```

OK，下面就要开始学习深度学习模型了。。。

> TensorFlow禁用没有使用sse编译的Warning，需添加环境变量 `export TF_CPP_MIN_LOG_LEVEL=2`
> 参考：https://github.com/tensorflow/tensorflow/issues/8037

-----

# 在docker容器中的TensorFlow环境

需要安装`nvidia-container-runtime`插件，才能正确运行支持GPU的容器。参考：[https://github.com/NVIDIA/nvidia-docker]。

>注意，安装过程会 **覆盖** `/etc/docker/daemon.json` 配置文件！需要提前备份。

安装步骤是：
```
curl -s -L https://nvidia.github.io/nvidia-docker/centos7/x86_64/nvidia-docker.repo | tee /etc/yum.repos.d/nvidia-docker.repo

yum install -y nvidia-docker2
pkill -SIGHUP dockerd
```

安装后的`/etc/docker/daemon.json` 如下（阿里云的仓库镜像是后来添加的）：
```
[root@n170 ~]# cat /etc/docker/daemon.json
{
    "registry-mirrors": ["https://lmigye0h.mirror.aliyuncs.com"],
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

其中增加了`nvidia-container-runtime`这个运行时插件，这是`nvidia-docker` v2 的实现方式了，运行一个容器验证一下：
```
nvidia-docker run --rm nvidia/cuda nvidia-smi
nvidia-docker run --rm -e NVIDIA_VISIBLE_DEVICES=1 nvidia/cuda nvidia-smi
```

其中环境变量`NVIDIA_VISIBLE_DEVICES`是指定GPU设备的可见性，可以是 0,1,... 这样逗号分隔的一个或多个GPU id，也可以是all或none。
参考：https://github.com/nvidia/nvidia-container-runtime#nvidia_visible_devices

其实`nvidia-docker`只是一个包装脚本，实际执行的命令是`docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi` 。

至于TensorFlow的容器，执行
`docker run --runtime=nvidia -ti --rm -p 8000:8888 -p 6006:6006 tensorflow/tensorflow:latest-gpu`

这个镜像有3.36GB。8888端口是jupyter notebook的，6006是tensorboard的端口，因为我的这台机器的8888端口被占用了，所以映射到了8000。
容器启动后，会输出jupyter notebook的访问token，在浏览器输入主机的IP（假设为2.2.2.170）和映射端口号（这里是8000，不是默认的8888），即
http://2.2.2.170:8000/?token=79f542ddc22fb567f3c2900c9310f1ce30847d5c5f927cba 
就会打开jupyter notebook，里面有三个TensorFlow入门介绍的ipynb文件，这样就可以编辑运行Python代码了。

```
[root@n170 ~]# docker run --runtime=nvidia -ti --rm -p 8000:8888 -p 6006:6006 tensorflow/tensorflow:latest-gpu
[I 03:08:12.136 NotebookApp] Writing notebook server cookie secret to /root/.local/share/jupyter/runtime/notebook_cookie_secret
[W 03:08:12.159 NotebookApp] WARNING: The notebook server is listening on all IP addresses and not using encryption. This is not recommended.
[I 03:08:12.165 NotebookApp] Serving notebooks from local directory: /notebooks
[I 03:08:12.165 NotebookApp] 0 active kernels
[I 03:08:12.165 NotebookApp] The Jupyter Notebook is running at:
[I 03:08:12.165 NotebookApp] http://[all ip addresses on your system]:8888/?token=79f542ddc22fb567f3c2900c9310f1ce30847d5c5f927cba
[I 03:08:12.165 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 03:08:12.166 NotebookApp] 
    
    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://localhost:8888/?token=79f542ddc22fb567f3c2900c9310f1ce30847d5c5f927cba

[root@n170 ~]# # 按 Ctrl + p, q 键退出容器交互终端，容器仍在后台运行
```

> Anaconda 中也有jupyter notebook，在主机执行命令`jupyter notebook`就会运行后台服务，并启动浏览器打开页面。
> 默认只能允许localhost访问，如果需要设置别的机器也可以通过主机的IP地址访问notebook，可以参考 [http://jupyter-notebook.readthedocs.io/en/stable/public_server.html] 。

> notebook中用matplotlib画图，如果不想写`plt.show()`，可以在代码前加上`%matplotlib inline`指令，这样执行`plt.plot(...)`就会输出图形。

# pip 和 cuda

如果不是按上面小节的步骤使用`conda`，就要按照教程的步骤，先安装cuda了。

> pip换源，在文件`$HOME/.pip/pip.conf`中添加
```
[global]
trusted-host =  mirrors.tuna.tsinghua.edu.cn
index-url = https://mirrors.tuna.tsinghua.edu.cn/pypi/simple
 ```

没有安装cuda，直接执行`pip install tensorflow-gpu`，取决于系统的Python版本，不论2.7，3.5或3.6版，都可以安装对应1.4.0版本，但执行上面的测试例子，就会报错：
```
... ...
ImportError: libcublas.so.8.0: cannot open shared object file: No such file or directory
... ...
```

就是说找不到cuda的动态库。

> 前面也提到了，其实cuda类似JDK，但Nvidia没有把cuda的动态库打包单独提供（类似JRE）。
> `conda`自己打包了需要的动态库（cudatoolkit，cudnn），可以一键安装，但`pip`就没有这么贴心了，需要安装完整版的cuda SDK。

需要下载的文件和具体安装步骤可见[https://developer.nvidia.com/cuda-downloads] ，目前Nvidia官网提供的是cuda 9.0（不知向前兼容性如何），旧版cuda的下载链接是[https://developer.nvidia.com/cuda-toolkit-archive] ，还要注册一下，然后下载并安装cuDNN的库。

如果之前没有安装显卡驱动的话，按上面官网的介绍，以为上面的步骤会把cuda 9.0和内核驱动一起安装上，而且确实安装了名为`nvidia-kmod`的包，但重启后执行`nvidia-smi`，发现并没有安装成功，不知是什么问题。
所以还是要按照更前面小节的步骤从elrepo安装`kmod-nvidia`包，不过安装过程会有包冲突，需要根据提示信息卸载：
```
sudo yum erase 1:nvidia-kmod-384.81-2.el7.x86_64
sudo yum erase 1:xorg-x11-drv-nvidia-384.81-1.el7.x86_64
sudo yum-config-manager --disable cuda-9-0-local
```

之后再重新执行安装`kmod-nvidia`的命令，重启后验证安装是否正确。再增加环境变量 
`export LD_LIBRARY_PATH=/usr/local/cuda/lib64/:$LD_LIBRARY_PATH` 
TensorFlow应该就可以找到需要的动态库了。

# 深度学习框架

参考：[https://en.wikipedia.org/wiki/Comparison_of_deep_learning_software]

比较常见的几个框架有：
+ Tensorflow : Google的项目，参考TensorFlow OSDI`2016的论文，设计目标是在大规模集群和异构硬件（GPU，TPU，ASIC等）上支持深度学习网络的训练和应用
+ MXNet：由[CMU的李沐博士](https://www.cs.cmu.edu/~muli/) ，[华盛顿大学的陈天齐博士](https://homes.cs.washington.edu/~tqchen/) 等开发的项目，他还有一篇博客介绍了[MXNet设计和实现](http://mli.github.io/2015/12/03/mxnet-overview/) 。目前是Apache的孵化项目，Amazon也在推广MXNet（李沐博士在Amazon工作）。在MXNet的基础上，他们还发布了[更灵活的前端Gluon（胶子）](http://mp.weixin.qq.com/s/_9aY-7aTZDOjeWFKntLnXA) 和[更可拓展的后端NNVM compiler](https://zhuanlan.zhihu.com/p/29914989)
+ Cognitive Toolkit（CNTK）：这是微软的深度学习项目
+ Theano：蒙特利尔大学MILA实验室开发的项目，2017年11月15日发布1.0版后就不再继续开发
+ PyTorch：是基于Lua的Torch项目的Python版本，由Facebook开发
+ Caffe2，Caffe：是由[UC Berkeley的贾扬清博士](http://daggerfs.com) 开发的，他已经在Facebook工作，所以Caffe2也是Facebook的一个项目
+ Keras：这个项目是对一些深度学习项目的更高层抽象和统一包装，官方支持的后端有TensorFlow，CNTK和Theano，有些深度学习项目也会提供对Keras的支持。当然，有的项目，像PyTorch，本身的抽象就比较高层，与Keras相当，另外像MXNet自己也有类似的前端Gluon。

从Wiki上的比较列表来看，对深度学习框架的关注点主要有：是否支持GPU加速，支持分布式集群，自动推导梯度，支持的网络类型（CNN，RNN等），是否有预先训练的模型等。
问了两位搞机器学习方向的同学，他们觉得TensorFlow偏底层，工程化，不如PyTorch写代码直观，Keras虽然理念很好，但性能上还差一点。他们目前还是用单机的GPU来训练模型，跑一次也要不少时间，但还没有准备搞TensorFlow那种分布式计算集群。

对DNN的了解太少了，要抓紧时间啊！
