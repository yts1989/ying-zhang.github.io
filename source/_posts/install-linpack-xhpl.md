title: 在MPI集群执行Linpack测试
date: 2017-04-12
category: [misc]
tags:

---
使用虚拟机搭建一个3台VM的CentOS 7集群，安装mpi，blas和hpl，执行linpack测试集群的计算性能。
尝试编译netlib hpl 2.2时遇到一个错误，导致不能使用mpi。
后来发现可以直接使用Intel编译好的xhpl。由于不理解mpi和xhpl的选项，集群的性能比单机还明显低得多，还没找到原因;-(
<!--more-->

<!-- TOC -->

- [安装和设置mpich，blas（atlas），hpl](#%E5%AE%89%E8%A3%85%E5%92%8C%E8%AE%BE%E7%BD%AEmpich%EF%BC%8Cblas%EF%BC%88atlas%EF%BC%89%EF%BC%8Chpl)
    - [设置mpich](#%E8%AE%BE%E7%BD%AEmpich)
    - [BLAS, LINPACK/LAPACK, HPL](#blas-linpacklapack-hpl)
    - [单机上执行 HPL](#%E5%8D%95%E6%9C%BA%E4%B8%8A%E6%89%A7%E8%A1%8C-hpl)
- [使用Intel MKL Benchmarks](#%E4%BD%BF%E7%94%A8intel-mkl-benchmarks)
- [Todo: 集群中测试mpi xhpl](#todo-%E9%9B%86%E7%BE%A4%E4%B8%AD%E6%B5%8B%E8%AF%95mpi-xhpl)

<!-- /TOC -->

# 安装和设置mpich，blas（atlas），hpl
编译可只在一台VM上进行，然后将编译的结果拷贝到其它VM。

## 设置mpich
参考 [MPI安装手册](http://www.mpich.org/static/downloads/3.2/mpich-3.2-installguide.pdf) 和 [MPI用户手册](http://www.mpich.org/static/downloads/3.2/mpich-3.2-userguide.pdf)
```
# 安装编译器
yum install -y gcc gcc-gfortran gcc-c++ bzip2 wget

# yum 可以安装mpich-3.2.x86_64，但只有lib，没有bin，所以这里从src编译
wget http://www.mpich.org/static/downloads/3.2/mpich-3.2.tar.gz
tar axf mpich-3.2.tar.gz
cd ~/mpich-3.2
./configure prefix=/opt/mpich
make -j 8 && make install  # make只会编译lib，make install才会编译lib和bin

cp -r examples/ /opt/mpich/

# 把编译结果打包
cd ~
tar zcf mpich-3.2-build.tar.gz /opt/mpich/
# 可以将其保存到主机上

echo 'export PATH=$PATH:/opt/mpich/bin'                         >> /etc/profile
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/mpich/lib'   >> /etc/profile
source /etc/profile

# 在本机测试一下
mpiexec -n 3 /opt/mpich/examples/cpi
```

mpiexec会在单个主机创建N个进程（通过 -n 指定）执行后面的命令（程序），如果通过 -f host_list_file 指定集群节点列表，会把进程分布在这些节点上分布执行。
mpiexec执行的可以是普通的命令，这时只是重复地执行N次，但这些命令之间并没有什么联系。
如果执行的是一个使用了mpi库的程序，那么程序执行中会彼此通信，协调计算进度，从而充分利用集群的计算资源。
要在集群上运行mpi程序，需要所有的节点上都有这个程序的可执行文件，以及需要的数据，配置文件，环境变量等。当然可以将这些文件等拷贝到各节点，但一般会创建一个共享目录，集群中的节点都将共享目录挂载到相同的路径，并将mpi程序及相关文件放到共享目录下。

## BLAS, LINPACK/LAPACK, HPL
[BLAS（Basic Linear Algebra Subprograms）- wiki](https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms) 或 [BLAS基础线性代数程序集 - wiki](https://zh.wikipedia.org/wiki/BLAS) 是一个API标准，有多个开源实现，如Netlib BLAS（Fortran实现），Netlib ATLAS，Intel MKL和ACML等。这里使用ATLAS。
```
wget https://downloads.sourceforge.net/project/math-atlas/Stable/3.10.3/atlas3.10.3.tar.bz2
tar axf atlas3.10.3.tar.bz2   # 需要已安装bzip2
cd ATLAS
mkdir build; cd build
../configure
make -j 8 && make install # 编译完成后安装到/usr/local/atlas，包括 lib 和 include
# 同样将编译的结果也打包保存
mkdir atlas
mv bin/ lib/ include/ atlas/
tar zcf atlas-build.tar.gz atlas/
```

[LINPACK](https://en.wikipedia.org/wiki/LINPACK)是一个线性代数数值计算库，其中用到了BLAS。不过目前多是使用它的后继[LAPACK](https://en.wikipedia.org/wiki/LAPACK)。[HPLinpack（Highly Parallel Computing benchmark，HP不是指惠普公司）](http://www.netlib.org/benchmark/hpl/)是一个使用LINPACK测试集群浮点计算性能的测试基准程序，测试的结果是多少GFPLOPS。

参考 http://blog.chinaunix.net/uid-20104120-id-4071017.html 。
```
wget http://www.netlib.org/benchmark/hpl/hpl-2.2.tar.gz
tar axf hpl-2.2.tar.gz
cd hpl-2.2
cp setup/Make.Linux_PII_CBLAS_gm Make.x86_64

# 编辑 Make.x86_64，修改的内容如下。
# 因为设置MPdir后有编译错误，所以没有设置。这样编译出来的是单机版的。
# 注意各值结尾不要有 空格。
# 虽然在 INCdir、BINdir和LIBdir删掉了$(ARCH)，但最终还创建了x86_64的空目录
ARCH         = x86_64

TOPdir       = $(HOME)/hpl-2.2
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin
LIBdir       = $(TOPdir)/lib

MPdir        =
MPinc        =
MPlib        =

LAdir        = /usr/local/atlas  # atlas 执行了make install之后的安装目录
LAinc        = -I$(LAdir)/include
LAlib        = $(LAdir)/lib/libcblas.a $(LAdir)/lib/libatlas.a

# 开始编译
make arch=x86_64 -j 8

# 也将编译的结果也打包保存
mv hpl hpl.bk.d; mkdir hpl
mv bin/ lib/ include/ hpl/
rmdir hpl/bin/x86_64/ hpl/include/x86_64/ hpl/lib/x86_64/
tar zcf hpl-build.tar.gz hpl/
```

## 单机上执行 HPL
```
cd /root/hpl-2.2/hpl/bin/
mpiexec -n 4 ./xhpl  # 得到的结果比较差，只有约0.4GFLOPS
# 因为需要配置HPL.dat，选择合适的参数，另外编译中一些选项也会有影响
```

# 使用Intel MKL Benchmarks
参考 https://software.intel.com/en-us/articles/intel-mkl-benchmarks-suite 。

```
wget http://registrationcenter-download.intel.com/akdlm/irc_nas/9752/l_mklb_p_2017.2.015.tgz
tar axf l_mklb_p_2017.2.015.tgz
cd l_mklb_p_2017.2.015/benchmarks_2017/linux/mkl/benchmarks/linpack
./runme_xeon64
```
测试的结果约 150GFLOPS。
还有一个Windows版的，在主机上测试也是接近的结果。

# Todo: 集群中测试mpi xhpl
直接运行 `l_mklb_p_2017.2.015/benchmarks_2017/linux/mkl/benchmarks/mp_linpack` 中的 `runme_intel64_static` 会报不识别 perhost 参数的错误。
执行 `mpiexec -f hosts -n 12 ./xhpl_intel64_static`（xhpl_intel64_static和HPL.dat拷贝到了/root，hosts是节点列表），在其它节点通过`top`监视，确实执行了xhpl_intel64_static，但输出只有当前机器的，而且只有约4GFLOPS（单机执行xhpl_intel64_static也是这么多，按理应接近4×集群节点数啊），不知道问题具体出在哪里，看来还是需要仔细看文档了。或许MPI，BLAS等全部使用Intel的版本？
参考
+ [Intel® MPI Library - Documentation](https://software.intel.com/zh-cn/articles/intel-mpi-library-documentation)
+ [Intel® Optimized MP LINPACK Benchmark for Clusters](https://software.intel.com/en-us/node/528457)
+ [HPL application note](https://software.intel.com/en-us/articles/performance-tools-for-software-developers-hpl-application-note)
+ [HPC LINPACK benchmark](http://khmel.org/?p=527)
+ [如何做LINPACK测试及性能优化](http://blog.sciencenet.cn/blog-935970-892936.html)
