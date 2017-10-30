title: 设置及使用HTTP代理
date: 2017-04-14
category: [misc]
tags:

---
记录一下设置Squid作为HTTP代理，及docker、yum、apt等的代理配置。
<!--more-->

<!-- TOC -->

- [使用Squid提供HTTP代理](#%E4%BD%BF%E7%94%A8squid%E6%8F%90%E4%BE%9Bhttp%E4%BB%A3%E7%90%86)
    - [主机上安装和设置Squid](#%E4%B8%BB%E6%9C%BA%E4%B8%8A%E5%AE%89%E8%A3%85%E5%92%8C%E8%AE%BE%E7%BD%AEsquid)
    - [以Docker容器的方式运行Squid](#%E4%BB%A5docker%E5%AE%B9%E5%99%A8%E7%9A%84%E6%96%B9%E5%BC%8F%E8%BF%90%E8%A1%8Csquid)
- [使用HTTP代理](#%E4%BD%BF%E7%94%A8http%E4%BB%A3%E7%90%86)
    - [全局的环境变量](#%E5%85%A8%E5%B1%80%E7%9A%84%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F)
    - [Docker](#docker)
    - [yum](#yum)
    - [apt](#apt)

<!-- /TOC -->

# 使用Squid提供HTTP代理

## 主机上安装和设置Squid
作为网关的`n147`机器，公网IP是`2.2.2.147`。安装Squid，然后修改配置，启用服务。
```
yum install -y squid # CentOS
apt install -y squid # Ubuntu
apk add squid     # Alpine

# squid的配置文件在 /etc/squid/squid.conf，修改内容可参考下面的 Dockerfile

# 修改配置后，初始化squid的工作目录
squid -z

# 启动服务
systemctl enable squid
systemctl start  squid
```

## 以Docker容器的方式运行Squid
Dockerfile内容如下：
```
FROM alpine:latest

RUN apk update --no-cache; \
    apk add squid --no-cache

# 可以在squid.conf中限制允许访问此代理的IP范围，否则只有内网IP可以访问
RUN sed  -i "/RFC 4291/a acl ics src 2.2.2.0/24" squid.conf; \
    sed  -i "/RFC 4291/a acl ics src 2.2.3.3/32" squid.conf

# 可以修改默认的端口号，如果修改了默认端口，需要修改下面的 EXPOSE 部分
RUN sed -i "/http_port/c http_port 8888" squid.conf

# 开启cache
RUN sed -i '/cache_dir/s/#//g' /etc/squid/squid.conf

# 或者直接使用修改过的配置文件
# ADD squid.conf /etc/squid/squid.conf

# squid -z用于初始化，创建cache目录，但直接在Dockerfile中
# RUN squid -z
# 却无法创建cache目录，导致squid无法启动
# 故将初始化和启动命令写入脚本中

RUN echo -e '#!/bin/sh\n[ -d /var/cache/squid/00 ] || squid -z\nsquid -N' >/squid.sh; \
    chmod +x /squid.sh

EXPOSE 3128
CMD ["/squid.sh"]
```

构造镜像：`docker build ./ -t squid:latest`
启动容器：`docker run -d -p 3128:3128 --name squid squid:latest`


# 使用HTTP代理
内网其它不能直接访问外网的机器可以设置使用`n147`提供的代理服务。

## 全局的环境变量
在`/etc/environment`（不需要`export`），`/etc/profile`或`/etc/profile.d/http_proxy.sh`导出`http_proxy`和`https_proxy`
```
export http_proxy=http://2.2.2.147:3128
export https_proxy=http://2.2.2.147:3128
```

`squid`可以作为https代理，只要设置 `https_proxy=http://2.2.2.147:3128`， 即这个环境变量以`http://`开头。

## Docker
Docker需要[单独设置代理](https://docs.docker.com/engine/admin/systemd/)，新建文件`/etc/systemd/system/docker.service.d/http-proxy.conf`，内容如下（注意多项环境变量之间要有空格，还设置了对私有镜像仓库不使用代理）：
```
[Service]
Environment="HTTP_PROXY=http://2.2.2.147:3128" "HTTPS_PROXY=http://2.2.2.147:3128"  "NO_PROXY=localhost,10.0.0.147"
```
重启docker daemon： `systemctl restart docker`，执行`docker info`查看是否生效。

## yum
yum 会使用全局代理设置，也可以单独设置代理，在`/etc/yum.conf`中增加：
```
proxy=http://2.2.2.147:3128
```

## apt
在文件`/etc/apt/apt.conf`中增加：
```
Acquire::http::proxy  "http://2.2.2.147:3128";
Acquire::https::proxy "http://2.2.2.147:3128";
```
