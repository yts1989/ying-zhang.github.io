
title: CentOS 7 安装支持认证的Mesos集群
category: [cloud]
tags: 
date: 2017-12-20
---

在CentOS 7上安装Mesos集群，设置对Slave和框架的认证，改为普通用户执行任务。Chronos在容器中执行GPU作业。

-----
<!--more-->

> 注意：
> 以下的设置都是以root用户权限执行的。
> 机器的操作系统是CentOS 7，机器名n5（`/etc/hostname`和`/etc/hosts`都设置了机器名），IP地址10.1.1.5 。
> 为了简便，Mesos Master和Slave在同一台机器上，Zookeeper也是单机运行模式。

# Mesos集群基本设置

## 安装JDK

先安装Open JDK 8
```
yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel

# 在/etc/profile末尾增加环境变量，下文会把Zookeeper安装到/opt/zookeeper
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export PATH=$PATH:$JAVA_HOME/bin:/opt/zookeeper/bin
```

## 安装Zookeeper
下载并设置Zookeeper，参考[https://zookeeper.apache.org/doc/r3.4.11/zookeeperStarted.html] 。
```
cd ~
curl -O http://mirrors.nju.edu.cn/apache/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
tar axf zookeeper-3.4.11.tar.gz -C /opt/
rm  zookeeper-3.4.11.tar.gz
mv  /opt/zookeeper-3.4.11/ /opt/zookeeper

# 因为只使用一台机器，zk设置了server.1=n5:2881:3881，myid为1
# 更多的机器需要分别修改 myid 和 server.<id>
cat > /opt/zookeeper/conf/zoo.cfg <<EOF
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
dataLogDir=/var/log/zookeeper
clientPort=2181
server.1=n5:2881:3881
EOF

mkdir /var/lib/zookeeper /var/log/zookeeper
echo  1 >/var/lib/zookeeper/myid
```

因为前面将`/opt/zookeeper/bin`加入了`$PATH`环境变量，所以可以直接输入下面的命令，
+ `zkServer.sh start`，启动Zookeeper。
+ `zkServer.sh status`，正常的话会输出包含`Mode: standalone`的信息，即处于单独运行模式。
+ `zkCli.sh -server n5:2181` 或 `zkCli.sh`，以进入Zookeeper的Shell，在其中查看和修改Zookeeper的值。

设置Zookeeper的Systemd服务配置文件，编辑文件 `/lib/systemd/system/zookeeper.service` ，内容如下：
```
[Unit]
Description=Apache Zookeeper
After=network.target

[Service]
Type=forking
User=root
Group=root
Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/bin/zkServer.sh restart

[Install]
WantedBy=multi-user.target
```

启动Zookeeper服务：`systemctl enable zookeeper.service; systemctl start zookeeper.service `
确认服务正常启动了：`zkServer.sh status`

## 安装Mesos，Marathon和Chronos

参考《Mesos实战》，通过Mesosphere的源安装Mesos，Marathon和Chronos。
由于Mesosphere（dc/os）修改了文档（可能是为了推广dc/os吧），现在的[Apache Mesos](https://mesos.apache.org)官方文档不太友好，好在安装包源还可以用。
> Ubuntu的源是 [http://repos.mesosphere.io/ubuntu]

```
# rpm -Uvh http://repos.mesosphere.io/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum install -y http://repos.mesosphere.io/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum install -y mesos marathon chronos haproxy
```

安装的版本分别是（2017-12-21）：Mesos 1.4.1，Marathon 1.5.4，Chronos 2.5.1

安装后，会创建 Mesos Master 和 Slave 的Systemd服务配置文件，还设置了`/etc/mesos/zk`文件内容为`zk://localhost:2181/mesos`，这正是Zookeeper的默认端口，就无需更改了。

对多个网卡的机器，如果要mesos使用某个特定的网卡，就需要在`/etc/default/mesos-master`中设置该网卡对应的IP地址，这里是`IP=10.1.1.5`。还可以在这个文件设置`HOSTNAME`（机器名）和`CLUSTER`（mesos集群名）。
其实也可以在这个文件设置`zk`，不过这只对Master有效。

在`/etc/default/mesos`这个文件的设置对master和slave都有效。

> 一些参数既可以直接作为启动命令的命令行参数，也可以作为环境变量写到上面提到的配置文件中。
> 这些配置文件的路径是硬编码在`/usr/bin/mesos-init-wrapper`这个启动脚本中的。

然后就可以启动服务了，为了简便，这里关闭了系统的防火墙：
```
systemctl disable firewalld; systemctl stop firewalld 
systemctl restart mesos-master mesos-slave chronos marathon
systemctl status  mesos-master mesos-slave chronos marathon

```

> Mesosphere的yum仓库中也有zookeeper，可执行`yum install -y mesosphere-zookeeper`安装，会安装到`/opt/mesosphere/zookeeper/bin/`，并生成`zookeeper.service`的systemd服务（当然需要配置server.id，并手动启用服务）。

通过（其它机器的）浏览器访问`http://10.1.1.5:5050`，应该就可以打开Mesos Web UI了，在Framworks中会列出Chronos，访问`http://10.1.1.5:4400`，可以打开Chronos Web UI。但**Marathon没有启动成功**。

## 处理Marathon服务启动问题
参考 [https://github.com/mesosphere/marathon] 。
修改Marathon的Systemd服务配置文件`/usr/lib/systemd/system/marathon.service`，主要是在启动命令增加了`master`和`zk`参数（注意参数与值之间是用空格分隔的，不要写成**等号“=”**；zk的路径是marathon，不是mesos），用户改为`root`，并改正了`mkdir`和`chmod`的完整路径。
完整内容如下：

```
[Unit]
Description=Scheduler for Apache Mesos
Requires=network.target

[Service]
Type=simple
WorkingDirectory=/usr/share/marathon
EnvironmentFile=/etc/default/marathon
Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk" 
ExecStart=/usr/share/marathon/bin/marathon      \
    --master n5:5050 --zk zk://n5:2181/marathon

ExecReload=/usr/bin/kill -HUP $MAINPID
Restart=always
RestartSec=60
SuccessExitStatus=
User=root
ExecStartPre=/usr/bin/mkdir -p /run/marathon
ExecStartPre=/usr/bin/chmod 755 /run/marathon
PermissionsStartOnly=true
LimitNOFILE=1024

[Install]
WantedBy=multi-user.target
```

保存上述设置文件后，执行`systemctl daemon-reload; systemctl restart marathon`重启服务，稍等一会儿，Mesos Web UI的Framworks列表中就有Marathon了。Marathon Web UI地址是`http://10.1.1.5:8080`。

# 清理Mesos集群
> 这节是为强迫症患者准备的。

清理运行任务记录
```
systemctl stop mesos-master mesos-slave
rm -rf /var/mesos /var/log/mesos
mkdir  /var/mesos /var/log/mesos

/opt/zookeeper/bin/zkCli.sh #进入zk的shell，执行下面的命令
rmr /mesos
rmr /chronos
rmr /marathon
quit # 退出zk的shell

systemctl start mesos-master mesos-slave
```

清理mesos下载的镜像文件
```
systemctl stop mesos-master mesos-slave
rm -rf /tmp/mesos
mkdir  /tmp/mesos

systemctl start mesos-master mesos-slave
```

# 支持GPU资源

## Mesos Slave的设置

> 当然，首先要有GPU硬件，且安装了硬件驱动。可参考[CentOS 7 安装TensorFlow GPU深度学习环境](/cloud/2017/setup-tensorflow-gpu-centos7/)。
> 参考：[http://mesos.apache.org/documentation/latest/gpu-support/]

对GPU的支持是Mesos Slave负责的，在`/etc/default/mesos-slave`中增加：
```
ISOLATION="gpu/nvidia,filesystem/linux,docker/runtime,cgroups/cpu,cgroups/mem,network/cni,cgroups/perf_event,posix/disk,cgroups/devices"
```

由于使用GPU的容器需要Nvidia提供的运行时插件，Mesos只支持自己的容器引擎加载GPU：它可以使用Docker的镜像文件，但运行时是Mesos实现的，而不是Docker。
由此导致容器中的默认环境变量没有包括Nvidia的可执行文件和动态库的路径，为了方便，在Slave设置一个默认的环境变量，同样是在`/etc/default/mesos-slave`中，增加：
```
executor_environment_variables=/etc/mesos/slave-executor-env.json
```

其中`/etc/mesos/slave-executor-env.json`是手动创建的文件，内容为：
```
{
"PATH":"/opt/anaconda3/bin:/usr/local/nvidia/bin:/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
"LD_LIBRARY_PATH":"/usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64",
"http_proxy": "http://n147:3128",
"https_proxy":"http://n147:3128",
"TZ":"GMT-8"
}
```

先停止Slave服务，删除旧的临时文件`rm -f /var/mesos/meta/slaves/latest`，
然后重启服务`systemctl restart mesos-slave`，刷新Mesos Web UI，应该就可以看到新增的GPU资源了。

## Marathon的设置
在Marathon的Systemd服务配置文件的`ExecStart`处增加参数，完整的启动命令是：
```
ExecStart=/usr/share/marathon/bin/marathon      \
    --master n5:5050 --zk zk://n5:2181/marathon \
    --enable_features gpu_resources
```

重启服务`systemctl daemon-reload; systemctl restart marathon`，可以在Marathon Web UI的about页面 [http://10.1.1.5:8080/ui/#/apps?modal=about] 确认启用了GPU。

## Chronos的设置

[Mesos官方版的Chronos](https://github.com/mesos/chronos) 还不支持GPU，我们使用一个修改过的Fork [https://github.com/reneploetz/chronos] ，从源码编译（需预先安装Maven）：
```
cd ~
git clone https://github.com/reneploetz/chronos.git

cd chronos
# 自带的 ./build-release.sh 脚本是在Docker容器中构建的，这里直接用Maven编译
# 需要预先安装Maven。构建的版本号是3.0.3。

curl --silent --location https://rpm.nodesource.com/setup_8.x | sudo bash -
yum install -y nodejs
mvn clean package -Dmaven.test.skip=true
```

下面停用从官方源安装的Chronos服务，从命令行启动支持GPU的Chronos（当然，也可以修改Chronos的Systemd服务配置文件，使用支持GPU的Chronos）：
```
systemctl disable chronos; systemctl stop chronos

nohup java -jar /root/chronos/target/chronos-3.0.3-SNAPSHOT.jar \
  --master zk://n5:2181/mesos --http_port=4400                  \
  --enable_features=gpu_resources >/dev/null                    & 
```

启动后再次打开Chronos Web UI，http://10.1.1.5:4400 ，发现与官方最新版的不一样了。
添加一个Scheduled Job，Job Name，Command都可以随便填。Web UI中不能指定GPU资源量，需要在Job的JSON配置文件中设置。创建Job后，修改其JSON配置文件，内容如下：
```
{
  "name": "host-gpu-test",
  "command": "nvidia-smi > /root/nvidia-smi-out.txt ; whoami; id; pwd",
  "shell": true,
  "executor": "",
  "executorFlags": "",
  "taskInfoData": "",
  "retries": 0,
  "owner": "",
  "ownerName": "",
  "description": "",
  "cpus": 0.1,
  "disk": 256,
  "mem": 128,
  "gpus": 1,
  "disabled": false,
  "softError": false,
  "dataProcessingJobType": false,
  "fetch": [],
  "uris": [],
  "environmentVariables": [],
  "arguments": [],
  "highPriority": false,
  "runAsUser": "root",
  "concurrent": false,
  "constraints": [],
  "schedule": "R1//P1Y",
  "scheduleTimeZone": ""
}
```

修改了Job设置后，在Web UI点击绿色的Run按钮执行Job。

+ 为使用GPU资源，设置`"gpus": 1`，机器上共安装了两个GPU，Mesos可以按整数个的粒度分配GPU。
+ Chronos针对的是定时周期作业，这里只需要执行一次，所以设置了`"retries": 0, "schedule": "R1//P1Y"`，即只重复1次，间隔1年，失败后重试0次。
+ 执行的命令是`"command": "nvidia-smi > /root/nvidia-smi-out.txt ; whoami; id; pwd"`。
+ 注意到`"runAsUser": "root"`，即用**主机上的root用户账号来运行这个Job**，将`nvidia-smi`的输出重定向到`/root/nvidia-smi-out.txt`，Job执行成功后，可以在Host查看这个文件，确认Chronos对GPU的支持运行正常。
+ 通过执行`whoami; id`也可以**确认是root账号**。
+ 但`pwd`输出的则是Mesos Slave创建的沙盒的完整路径，看来是没有`chroot`。

要查看任务输出到终端的内容，
+ 需要在Mesos Web UI的Frameworks列表点击Chronos的ID，
+ 然后在Completed Tasks中选择任务ID对应的Sandbox链接，
+ 再打开`stdout`或`stderr`的链接。

> Windows 上用 Chrome v63 打开 Edit Job 的界面，编辑光标总是 **错位**，但在MacOS的Chrome则正常。。。
> 所以先用 VS Code 编辑好再粘贴过去吧。

## 使用Chronos的RESTful API
在客户端，将上面的JSON作业配置示例保存为 `dlkit.json`（注意，需修改 `name` ，否则会 **覆盖** 同名作业的配置），然后使用 `curl` 提交`POST`请求：
```
curl -iL -H "Content-Type: application/json" -d @dlkit.json n5:4400/v1/scheduler/iso8601
```

提交后会返回 http/1.1 204 No Content ，这时从Chronos Web UI可以看到添加的作业。
由于设置的 schedule 是未来的时刻，作业需要执行下面的命令手动启动（注意，链接的参数是 作业名）：
```
curl -iL -X PUT n5:4400/v1/scheduler/job/host-gpu-test
```
提交后会返回 http/1.1 204 No Content ，这时从Chronos Web UI可以看到作业的状态已经转为 QUEUED 或 RUNNING。

参数`-H "Authorization: Basic <Base64 Code>" ` 可以添加Base64编码的Chronos Basic验证方式的用户名密码，设置方法见下面小节。
也可使用 `--user <name>:<password>`。如，获取当前作业列表：
```
curl -s --user <name>:<password> n5:4400/v1/scheduler/jobs | jq
```

`jq` 是JSON格式化工具，可通过 `sudo apt install jq` 安装。

更多Chronos Restful API可参考：https://mesos.github.io/chronos/docs/api.html
Mesos Restful API 可参看：http://mesos.apache.org/documentation/latest/endpoints/ 

# 对Slave，Framwork的验证

## Mesos Master的设置
> 参考：
> http://mesos.readthedocs.io/en/latest/authentication/
> http://mesos.readthedocs.io/en/latest/authorization/ 

注意到Chronos的Job是以root账号执行的，可以直接执行Host的命令，操作Host上的文件路径。

下面设置开启对Slave和Framwork的验证，并让Framwork使用低权限的账号，禁止使用root账号。
在`/etc/default/mesos-master`中增加下面的环境变量：
```
authenticate=true
authenticate_slaves=true
credentials=/etc/mesos/master-credentials.json
acls=/etc/mesos/master-acls.json
```

`/etc/mesos/master-credentials.json`是允许接入Mesos集群的账号和密码（这个与Host的账号系统无关），内容如下：
```
{       
  "credentials":[
    {
      "principal":"MesosPrincipal",
      "secret":"f0e4c2f76c58916ec258f246851bea091d14d4247a2fc3e186"
    }
  ]
}
```

文件中，
+ `"principal":"MesosPrincipal"`是用户名，
+ `"secret":"...."`是**明文的密码**，是用`sha256sum`或`openssl rand -hex 32`计算的一个字符串的哈希值。当然，随便设置的密码都可以，但其中不能有英文的冒号“:”。

Slave和Framwork都可以用这个用户名密码。
可以在Master设置为不同的Slave和Framwork使用不同的用户名密码，这里为了简便，只使用这一组。

`/etc/mesos/master-acls.json`是访问控制列表，与Host的账号及上面设置的Principal都相关，内容如下：
```
{       
  "run_tasks":[
    {
      "principals":{"values":["MesosPrincipal"]},
      "users":{"values":["mesos"]}
    },
    {
      "principals":{"type":"NONE"},
      "users":{"values":["root"]}
    }
  ]
}
```

这个文件将Host的root账号映射到NONE，从而禁止以root账号执行命令；并将MesosPrincipal映射到Host的mesos账号，我们需要在主机上创建该账号：
```
groupadd -g 1000 mesos

# 对允许mesos账号ssh登录，并且可以使用scp，sftp的机器。添加用户后还需要passwd mesos设置密码
useradd -d /home/mesos -s /bin/bash  -u 1000 -g 1000 mesos

# 对禁止mesos账号ssh登录的机器。没有设置密码。
useradd -d /dev/null -s /sbin/nologin -u 1000 -g 1000 mesos
```
因为要在集群的**多个机器上使用相同账号**，并且还要在**容器中创建相同的账号**，需要保证mesos账号的UID和GID在各机器上是相同的。

> 集群的账号管理，应该考虑使用LDAP了。。。

重启Mesos Master服务：`systemctl restart mesos-master`。因为没有设置Slave登录的用户名密码，所以这时Slave离线了。

## Mesos Slave的设置
在`/etc/default/mesos-slave`文件添加下面的内容：
```
credential=/etc/mesos/slave-credential-pair
```

其中`/etc/mesos/slave-credential-pair`是登录到Master的用户名密码，这里只使用了MesosPrincipal这一个账号。
> 注意：按照官方文档的说法，这个账号文件不能以空行结尾，即不能使用VIM等编辑器编辑。
> 而且格式与Master的不同，因为Master可以保存多组用户名密码，但Salve只需要提供一组就可以了。
> 这里使用`echo`写入值。

```
echo -n "MesosPrincipal f0e4c2f76c58916ec258f246851bea091d14d4247a2fc3e186" > /etc/mesos/slave-credential-pair
```

重启Mesos Slave服务：`systemctl restart mesos-slave`，这时在Mesos Web UI应该就可以看到恢复上线的Slave了。

## Marathon的设置

> 参考：https://mesosphere.github.io/marathon/docs/framework-authentication.html

在Marathon的Systemd服务配置文件的`ExecStart`处增加参数，完整的启动命令是：
```
ExecStart=/usr/share/marathon/bin/marathon          \
    --master n5:5050 --zk zk://n5:2181/marathon     \
    --enable_features gpu_resources                 \
    --mesos_authentication                          \
    --mesos_authentication_principal MesosPrincipal \
    --mesos_authentication_secret_file /etc/mesos/credential-secret-only
```

其中文件`/etc/mesos/credential-secret-only`只包含密码：
```
echo -n f0e4c2f76c58916ec258f246851bea091d14d4247a2fc3e186 > /etc/mesos/credential-secret-only
```

重启服务`systemctl daemon-reload; systemctl restart marathon`，之后可以正常打开Marathon Web UI [http://10.1.1.5:8080/] 。
还可以在Mesos Web UI的Framworks中看到使用的Principal是MesosPrincipal。

## Chronos的设置

> 参考：https://mesos.github.io/chronos/docs/configuration.html

从命令行启动支持GPU的Chronos：
```
nohup java -jar /root/chronos/target/chronos-3.0.3-SNAPSHOT.jar         \
  --master=zk://n5:2181/mesos --http_port=4400                          \
  --enable_features=gpu_resources                                       \
  --mesos_authentication_principal=MesosPrincipal                       \
  --mesos_authentication_secret_file=/etc/mesos/credential-secret-only  \
  --http_credentials="ChronosUser:SomePassword" >/dev/null              &
```

其中使用的`/etc/mesos/credential-secret-only`跟Marathon的是同一个文件。
还设置了登录Chronos Web UI的用户名密码`ChronosUser:SomePassword`。这个只是针对Chronos的，而且是HTTP Basic验证，聊胜于无吧。

之后再以root账号执行Chronos的作业，将会一直显示`Queued`。
将用户改为`"runAsUser": "mesos"`才会正常执行，而且需要root权限的命令会在`stderr`输出权限错误的信息。

# 扩展到集群
集群中可以设置多个Zookeeper节点，多个Master节点，以达到高可用（需是单数个节点）。
需要在每个机器上创建mesos这个账号，并保证UID，GID与其它机器相同，并将Master或Slave相关的配置文件等拷贝到对应的机器上。
Marathon和Chronos框架只需在某一台机器上启动即可，也可以启动多个实例，设置高可用模式。

# 文件服务器设置
为了方便其它用户使用mesos账号拷贝文件到集群，需要搭建一个文件服务器。
集群已经把所有机器的硬盘加入了[Gluster分布式文件系统](https://wiki.centos.org/SpecialInterestGroup/Storage/gluster-Quickstart)，路径是`/gluster/volume2`，在其中新建文件夹`/gluster/volume2/data`作为用户上传文件的工作目录。
所有机器上都能同步看到这个相同的文件路径。

> GlusterFS有Linux系统的Native Client，可以[使用SSL/TLS安全验证](https://docs.gluster.org/en/latest/Administrator%20Guide/SSL/) 。由于不支持MacOS和Windows，且设置不方便，故不使用这种方式。

## SFTP 【X】
本来想允许某台机器的mesos账号ssh登录，这样也就默认开启了scp和sftp，可以使用Filezilla等FTP工具传文件。
但是，又想限制SFTP只能读写自己的`$HOME`，比如把mesos的`$HOME`设置为`/gluster/volume2/data`，只能读写这个文件夹。
OpenSSH确实可以[通过`chroot`限制某个组的用户 **只能读** 它的$HOME（或某个特定的文件夹）](https://bensmann.no/restrict-sftp-users-to-home-folder) 。
又来了但是，`chroot`后，对这个用户而言，整个文件系统就只有`$HOME`下的那些文件了，没有`/bin`、`/usr`等，也就无法执行`ssh`或者`scp`；而`sftp`还可以用，是因为设置中改用了OpenSSH内置的sftp功能，而不是外部的sftp程序（ `/usr/libexec/openssh/sftp-server`）；
而且必须把`$HOME`的owner设置为`root`，权限只能是`755`或`750`，就是说即便把owner group改为mesos，它也只能对自己的`$HOME`有读权限，但 **不能写**；

虽然可以在`$HOME`新建子目录用于mesos读写，但感觉还是不爽，试试别的方案。

## NFS on Gluster 【X】
NFS是 *nix 系统上的文件共享服务。Linux，Windows和MacOS也都支持挂载NFS。问题是[需要额外的组件（Kerberos）才能支持用户登录验证](http://joshuawise.com/kerberos-nfs)，否则是公开访问的！

Gluster有专门的NFS组件`nfs-ganesha`，比使用系统默认的NFS组件好一点。不使用Kerberos的`nfs-ganesha`设置过程如下：
> 参考：
+ [Manually Configuring nfs-ganesha Exports - Red Hat document](https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.1/html/Administration_Guide/gluster-nfs_and_kernel-nfs_services.html)
+ [Configuring NFS-Ganesha over GlusterFS](http://docs.gluster.org/en/latest/Administrator%20Guide/NFS-Ganesha%20GlusterFS%20Integration/)

```
yum install -y glusterfs-ganesha nfs-ganesha nfs-ganesha-gluster
/usr/libexec/ganesha/ganesha.nfsd -N NIV_MAJ  # 修改Log级别，减少日志量
```

修改配置文件`/etc/ganesha/ganesha.conf`，其中：
+ Gluster卷的路径是`/gluster/volume2`，共享的是其子目录`/gluster/volume2/data`；
+ 映射到的 NFS 路径是`10.1.1.5:/data`，所有客户端都映射到 mesos 账号的UID 1000和GID 1000；
+ NFS 的版本是v3，没有用v4。

完整内容如下：
```
EXPORT{
      Export_Id = 1;
      Path = "/data";
      FSAL {
           name = GLUSTER;
           hostname="localhost";
           volume="volume2";
           volpath="/data";
           }
      Access_type = RW;
      Disable_ACL = true;
      Squash="All_Anonymous";
      Pseudo="/data";
      Protocols = "3";
      Transports = "UDP","TCP";
      SecType = "sys";
      
      anonymous_uid = 1000;
      anonymous_gid = 1000;
      }

NFS_Core_Param {
    NSM_Use_Caller_Name = true;
    Clustered = false;
    Rquota_Port = 875;
}
```

继续执行
```
gluster volume set volume2 features.cache-invalidation on
systemctl restart nfs-ganesha
```

执行`showmount -e localhost`查看是否正常，输出中应包含 `/data (everyone)`

在客户端挂载NFS到`/mnt` ： `sudo mount -t nfs 10.1.1.5:/data /mnt`，卸载`sudo umount -f /mnt`。

NFSv4的话，如果客户端与服务器的域名不一致，会把用户映射为`nobody`和`nogroup`；
NFSv3则会根据服务器的UID和GID，在客户端找有没有对应的账号：比如服务端用的mesos账号，UID是1000，碰巧客户端存在UID 1000的账号sosem，那就会显示sosem，没有找到对应的就直接显示UID或GID。
如果服务端的**匿名用户对共享文件有读写权限**，客户端总是可以在本机执行`sudo`命令随意读写远程的NFS文件，因为客户端的操作最终都是以匿名用户映射的账号在服务器执行的。
而为了能上传文件，就得开放写权限。
总之，NFS本身的安全机制实在是鸡肋，另选其它方案吧。

## Samba SMB/CIFS文件共享
Samba 文件共享非常方便（参考[局域网的远程操作](/misc/2016/remote/)对应小节），
+ Samba服务的配置很方便，自带用户验证，可以方便地设置共享目录、权限、用户（独立的密码）；
+ 映射成网络驱动器就跟读写本地硬盘体验一样；
+ Linux、MacOS、Windows都自带了客户端，可以直接挂载，Android手机也有ES文件浏览器App支持挂载。

但是，2017年5月份的勒索病毒WanaCrypt会通过445这个文件共享的默认端口攻击Windows机器，于是 **网络管理员禁止了445端口**。
好在可以修改Samba服务的默认端口，绕过封锁。Linux、MacOS和Android的ES文件浏览器都支持修改端口号。
但是，除了Windows只能使用445端口，无法修改。

设置步骤在[局域网的远程操作](/misc/2016/remote/)对应小节介绍过，这里再简单重复一下：
```
yum install -y samba
```

修改 `/etc/samba/smb.conf`，完整的内容如下：
```
# Run 'testparm' to verify the config is correct after you modified it.
[global]
  workgroup = SAMBA
  security = user
  passdb backend = tdbsam
  printing = cups
  printcap name = cups
  load printers = no
  cups options = raw
  smb ports = 4455

[data]
  comment = mesos work path
  path = /gluster/volume2/data
  writeable = yes
  create mask = 0664
  directory mask = 0775
; browseable = yes # 在Windows资源管理器不可见，只能通过输入完整的地址来访问
  valid users = mesos
```

然后设置mesos在Samba系统的密码：`smbpasswd -a mesos`。
之后启动smbd服务：`systemctl enable smb; systemctl start smb`。

Linux客户端需要执行`yum install -y samba-client cifs-utils` 安装必要的软件。
以挂载到`/mnt`为例，命令是
```
sudo mount -t cifs //10.1.1.5/data /mnt -o user=mesos,pass=MesosSambaPassword,port=4455,rw,iocharset=utf8
```

或写入`/etc/fstab`末尾，
```
//10.1.1.5/data /mnt cifs user=mesos,pass=MesosSambaPassword,port=4455,rw,iocharset=utf8 0 0
```

这样系统启动后就会自动挂载了，或执行`sudo mount -a`手动挂载`/etc/fstab` 。

可以在Linux桌面的文件管理器**地址栏**，或MacOS Finder的 **连接服务器 Command+K** 对话框输入`smb://10.1.1.5:4455/data`来访问Samba共享文件夹（会弹出账号密码窗口）。
Windows的地址格式是`\\10.1.1.5\data`，但由于不支持非445的端口号，所以无法访问。。。

## FTP
Windows和Linux客户端可以通过普通的FTP来上传文件，文件管理器可以直接打开FTP地址，也可以安装Filezilla。
MacOS不能直接在Finder（访达）打开FTP站点，需要安装Filezilla。

服务端，在CentOS服务器执行`yum install -y vsftpd`，配置文件是`/etc/vsftpd/vsftpd.conf`，内容如下：
```
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list
allow_writeable_chroot=YES
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
```

其中限制用户只能读写自己的`$HOME`。执行`touch /etc/vsftpd/chroot_list`建一个空的占位文件。
需要设置Host的mesos账号：
+ `$HOME`设置为`/gluster/volume2/data`，并设置为owner，有读写权限;
+ 还要设置`mesos`账号的密码 `passwd mesos`。

FTP是不加密的，使用主机上的账号和密码登录，可以写在地址里，即
`ftp://mesos:MesosHostPassword@10.1.1.5`，
或 
`ftp://mesos@10.1.1.5`，在弹出的窗口输入密码。

# Chronos在容器中执行GPU作业

前面使用Chronos提交了基于Host命令的作业，更好的办法是把运行环境打包成容器，以便于部署和管理。
我们创建了一个[`icsnju/dlkit`的Docker镜像](https://github.com/icsnju/dlkit/blob/master/Dockerfile)，然而由于网络原因，还有镜像本身太大了，在Docker Hub上没有构建成功，后来是在VPS上构建的，再Push到我们用[Harbor](https://github.com/vmware/harbor)搭建的本地镜像仓库。

为了配合Mesos使用低权限的mesos账号执行任务，需要在Docker镜像中也添加同名，同UID和GID的mesos用户，将其设置为可免输密码执行`sudo`命令，并将默认用户切换为mesos。
Dockerfile内容如下，构建出的镜像tag设为`local/dlkit:mesos`，下面会用到这个镜像。
```
FROM local/dlkit:latest
RUN  echo 'deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse' > /etc/apt/sources.list ; \
     rm /etc/apt/sources.list.d/*      ; \
     apt-get update                    ; \
     apt-get install -y sudo           ; \
     apt-get clean; apt-get autoremove ; \
     rm -rf /var/lib/apt/lists/*       ; \
     groupadd -g 1000 mesos            ; \
     useradd  -m -u 1000 -g 1000 mesos ; \
     echo "mesos ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/mesos

USER mesos
```

下面使用[Keras的MNIST CNN示例](https://github.com/keras-team/keras/blob/master/examples/mnist_cnn.py)
Chronos提交GPU的容器作业配置文件内容如下：
```
{
  "name": "mnist-cnn-demo",
  "command": "cd /data/mnist ; env; python mnist_cnn.py | tee out-`date +%Y%m%dT%H%M%S`.txt",
  "shell": true,
  "executor": "",
  "executorFlags": "",
  "taskInfoData": "",
  "retries": 0,
  "owner": "",
  "ownerName": "",
  "description": "",
  "cpus": 10,
  "disk": 1000,
  "mem": 10240,
  "gpus": 1,
  "disabled": false,
  "softError": false,
  "dataProcessingJobType": false,
  "fetch": [],
  "uris": [],
  "environmentVariables": [],
  "arguments": [],
  "highPriority": false,
  "runAsUser": "mesos",
  "concurrent": false,
  "container": {
    "type": "MESOS",
    "image": "local/dlkit:mesos",
    "network": "BRIDGE",
    "networkInfos": [],
    "volumes": [
      {
        "hostPath": "/gluster/volume2/data",
        "containerPath": "/data",
        "mode": "RW"
      }
    ],
    "forcePullImage": false,
    "parameters": []
  },
  "constraints": [],
  "schedule": "R1//P1Y",
  "scheduleTimeZone": ""
}
```

其中，
+ ``"command": "cd /data/mnist ; env; python mnist_cnn.py | tee out-`date +%Y%m%dT%H%M%S`.txt"`` 为便于排错，用`env; pwd`来输出环境信息，实际计算结果则重定向到``result`date +%Y%m%dT%H%M%S`.txt"``文件中；
+ `"cpus": 10,  "disk": 1000,  "mem": 10240,  "gpus": 1`，这是为容器分配的资源，如果机器上有2个GPU，那么最多可以申请2个，申请更多的话，Job会一直排队等待资源Offer，无法执行。CPU和内存资源也要多申请一些，防止OOM；
+ `"runAsUser": "mesos"`，以容器内的mesos账号执行命令；
+ `container`一节设置了使用的镜像，类型必须写成`MESOS`，而不能是`DOCKER`；
+ 还要在`volumes`中设置Host上传文件的路径到容器路径的映射，路径名中不能有`-`，否则无法挂载，导致容器无法启动。

提交后就可以在容器中执行GPU作业了。

> 在容器中执行任务已经支持了运行时的隔离，但文件服务使用的是同一个目录，没有隔离。
> 其实只要额外开发一个提交任务的页面，不让用户设置路径名，然后为每个用户设置不同的文件路径，就可以支持多用户的隔离了。


> PS，Mesos是Docker之前开发的，之后也没有充分利用Docker的隔离能力，设置上有点麻烦。