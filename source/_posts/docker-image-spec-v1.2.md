title: 【译文】Docker镜像格式规范，v1.2
category: [cloud]
tags:
date: 2017-05-16
---
原文见 https://github.com/moby/moby/blob/master/image/spec/v1.2.md
Docker已经迁移到Moby项目了。

<!--more-->
<!-- TOC -->

    - [title: 【译文】Docker镜像格式规范，v1.2](#title-%E3%80%90%E8%AF%91%E6%96%87%E3%80%91docker%E9%95%9C%E5%83%8F%E6%A0%BC%E5%BC%8F%E8%A7%84%E8%8C%83%EF%BC%8Cv12)
- [Docker镜像规范v1.2.0](#docker%E9%95%9C%E5%83%8F%E8%A7%84%E8%8C%83v120)
    - [术语](#%E6%9C%AF%E8%AF%AD)
        - [层（Layer）](#%E5%B1%82%EF%BC%88layer%EF%BC%89)
        - [镜像的JSON描述文件](#%E9%95%9C%E5%83%8F%E7%9A%84json%E6%8F%8F%E8%BF%B0%E6%96%87%E4%BB%B6)
        - [镜像文件变更集（changeset）](#%E9%95%9C%E5%83%8F%E6%96%87%E4%BB%B6%E5%8F%98%E6%9B%B4%E9%9B%86%EF%BC%88changeset%EF%BC%89)
        - [层的DiffID](#%E5%B1%82%E7%9A%84diffid)
        - [层的ChainID](#%E5%B1%82%E7%9A%84chainid)
        - [镜像的ImageID](#%E9%95%9C%E5%83%8F%E7%9A%84imageid)
        - [标签（Tag）](#%E6%A0%87%E7%AD%BE%EF%BC%88tag%EF%BC%89)
        - [镜像名（Repository）](#%E9%95%9C%E5%83%8F%E5%90%8D%EF%BC%88repository%EF%BC%89)
    - [镜像的JSON描述文件示例](#%E9%95%9C%E5%83%8F%E7%9A%84json%E6%8F%8F%E8%BF%B0%E6%96%87%E4%BB%B6%E7%A4%BA%E4%BE%8B)
    - [镜像的JSON描述文件说明](#%E9%95%9C%E5%83%8F%E7%9A%84json%E6%8F%8F%E8%BF%B0%E6%96%87%E4%BB%B6%E8%AF%B4%E6%98%8E)
        - [created](#created)
        - [author](#author)
        - [architecture](#architecture)
        - [os](#os)
        - [config](#config)
            - [User](#user)
            - [Memory](#memory)
            - [MemorySwap](#memoryswap)
            - [CpuShares](#cpushares)
            - [ExposedPorts](#exposedports)
            - [Env](#env)
            - [Entrypoint](#entrypoint)
            - [Cmd](#cmd)
            - [Healthcheck](#healthcheck)
                - [Test](#test)
                - [Volumes](#volumes)
                - [WorkingDir](#workingdir)
                - [rootfs](#rootfs)
        - [history](#history)
    - [创建镜像文件变更集](#%E5%88%9B%E5%BB%BA%E9%95%9C%E5%83%8F%E6%96%87%E4%BB%B6%E5%8F%98%E6%9B%B4%E9%9B%86)
    - [镜像的组合格式](#%E9%95%9C%E5%83%8F%E7%9A%84%E7%BB%84%E5%90%88%E6%A0%BC%E5%BC%8F)
- [TODO](#todo)
- [补充](#%E8%A1%A5%E5%85%85)
    - [alpine镜像的主机存储布局](#alpine%E9%95%9C%E5%83%8F%E7%9A%84%E4%B8%BB%E6%9C%BA%E5%AD%98%E5%82%A8%E5%B8%83%E5%B1%80)

<!-- /TOC -->

# Docker镜像规范v1.2.0

**镜像（Image）**是在基础文件集（root filesystem）之上依次变更的集合，及在容器运行的默认执行参数。本规范概述这些文件变更及执行参数的格式，创建和使用它们的方法。
此版本的镜像规范自Docker 1.12开始采用。

> 译注：本规范中的 **filesystem** 并非通常意义的文件系统，实际上只是一组文件及文件夹的集合，故译为 **文件集**。

## 术语
本规范使用以下术语:
### 层（Layer）
镜像由 **层** 组成。 每一层都是若干文件的变更集合。层不包含环境变量或默认参数等元数据。这些元数据是镜像整体的属性，而不是特定层的。

### 镜像的JSON描述文件
整个镜像有一个JSON描述文件，它包含镜像的基本信息，如创建的日期、作者、父镜像的ID、以及启动/运行的配置（如入口命令、默认参数、CPU / 内存份额、网络和数据卷等）。JSON文件还列出了组成镜像的每个层的加密散列及其命令历史。
该JSON文件是不可变的，更改它会导致重新计算整个镜像的ImageID（译注：见下面ImageID的计算方法小节），这意味着派生出一个新的镜像，而不是改变现有的镜像。
 
### 镜像文件变更集（changeset）
每个层都是一组在其父层之上增加、修改或删除文件的归档。使用分层或联合文件系统（如AUFS），或通过从文件系统快照计算差异（Diff），可以将一系列的层（即文件变更集）合并成一个虚拟的单层文件集合（one cohesive filesystem）。

### 层的DiffID
将一个层的所有文件内容序列化后，计算出一个加密散列来作为该层的标识。具体是将层打包为一个`tar`包，然后计算其SHA256摘要，用十六进制编码表示长度为256比特的串（共64个字符），
如`sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9`。
层的打包和解包必须是可重复的，以免更改层的ID，例如，应使用`tar-split`来保存tar包的header。注意，层的ID是基于未压缩的tar包计算的。

>译注：关于层的打包和解包的可重复性，`tar`程序将一组文件打包的 **顺序** 是与文件系统相关的，此处没有详细说明可重复性的实现方式，可参考Docker源码深入了解。
参考：[tar打包的顺序](https://unix.stackexchange.com/questions/120143/how-is-the-order-in-which-tar-works-on-files-determined)。

### 层的ChainID
为方便起见，可以给一串有序的层计算出一个ID，称为 **ChainID**。
仅有一个层时，其ChainID与该层的DiffID相同。多个层时，其ChainID由下面的递归公式给出：

$$ChainID(layerN)=SHA256hex(ChainID(layer(N-1))+ " \quad" +DiffID(layerN))$$ 

### 镜像的ImageID
每个镜像的ID是其JSON描述文件的SHA256散列值，用十六进制编码表示，
如`sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9`。
由于JSON文件包含镜像所有层的散列ID，据此计算出的ImageID，使得可以对镜像的即各层按内容寻址（Content Addressable，地址即各层的DiffID）。

### 标签（Tag）
Tag是用户为ImageID指定的说明文字。Tag中的字符只能是大小写英文字母、数字、短线、下划线和点，即`[a-zA-Z0-9_.-]`，首个字符不能是`.`或`-`。Tag不能超过127个字符。

### 镜像名（Repository）
这里的`Repository`是指镜像全名在冒号`:`之前的部分，冒号`:`之后的部分是镜像的标签（tag），用来区分镜像的版本。 如名为`my-app:3.1.4`的镜像，`my-app`就是镜像的 Repository 部分。
Repository又可以用斜杠`/`分隔开，`/`之前的部分是可选的DNS格式的主机名。主机名必须符合DNS规则，但 **不得** 包含下划线`_`字符，主机名可以有如`：8080`格式的端口号。
镜像名可以包含小写字符，数字和分隔符。 分隔符是句点`.`，一个或两个下划线`_`，或一个或多个短横线`-`，镜像名 **不允许** 以分隔符开头或结尾。

> 译注：
+ 这里的 Repository 容易与git的 **代码仓库** 概念混淆。
+ DNS名和主机名的格式稍有不同，一般来说，主机名不允许使用下划线`_`，参考[RFC 1123](https://tools.ietf.org/html/rfc1123)


## 镜像的JSON描述文件示例
下面是一个镜像的JSON描述文件示例：

```
{
  "architecture": "amd64",
  "author": "Alyssa P. Hacker &ltalyspdev@example.com&gt",
  "config": {
    "Cmd": [
      "--foreground",
      "--config",
      "/etc/my-app.d/default.cfg"
    ],
    "CpuShares": 8,
    "Entrypoint": [
      "/bin/my-app-binary"
    ],
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "FOO=docker_is_a_really",
      "BAR=great_tool_you_know"
    ],
    "ExposedPorts": {
      "8080/tcp": {}
    },
    "Memory": 2048,
    "MemorySwap": 4096,
    "User": "alice",
    "Volumes": {
      "/var/job-result-data": {},
      "/var/log/my-app-logs": {}
    },
    "WorkingDir": "/home/alice"
  },
  "created": "2015-10-31T22:22:56.015925234Z",
  "history": [
    {
      "created": "2015-10-31T22:22:54.690851953Z",
      "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
    },
    {
      "created": "2015-10-31T22:22:55.613815829Z",
      "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "diff_ids": [
      "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
      "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
    ],
    "type": "layers"
  }
}

```

注意，Docker生成的镜像JSON描述文件不包含为了格式化而插入的空格，这里是为了方便阅读。

## 镜像的JSON描述文件说明

### created
`string`
镜像创建的日期和时间，[ISO-8601格式](https://zh.wikipedia.org/wiki/ISO_8601)。

### author
`string`
创建和负责维护改镜像的人员或组织名，或Email。
### architecture
`string`
镜像中可执行文件的CPU架构，可以是 
+ `386`
+ `amd64`
+ `arm`
未来可能会支持更多的架构，有的容器引擎可能不支持某些架构。

### os
`string`
镜像运行的操作系统名，可以是 
+ `darwin`
+ `freebsd`
+ `linux`
未来可能会支持更多的架构，有的容器引擎可能不支持某些操作系统。

### config
`struct`
`config`结构是从镜像创建容器时，使用的基本执行参数。`config`可以是空值`null`，则创建容器时必须提供所有必要的执行参数。

`config`结构的各字段说明
#### User 
`string`
容器中进程执行使用的用户名或UID。如果创建容器时没有在命令行给出，将使用此配置项的值。下面的格式都是有效的：
+ `user`
+ `uid`
+ `user:group`
+ `uid:gid`
+ `uid:group`
+ `user:gid`

如果没有给出组名 `group`/`gid`，默认的组使用容器中`/etc/passwd`文件对应的`user`/`uid`项。

#### Memory 
`integer`
内存限值（以 **字节** 为单位）。如果创建容器时没有在命令行给出，则使用此配置项的值。

#### MemorySwap 
`integer`
总的内存使用量（内存 + swap），设置为`-1`则禁用 swap。如果创建容器时没有在命令行给出，则使用此配置项的值。

#### CpuShares 
`integer`
CPU份额（相对其它容器的权重）。如果创建容器时没有在命令行给出，则使用此配置项的值。

#### ExposedPorts 
`struct`
基于此镜像创建的容器公开的端口列表。此JSON结构的特殊之处在于它是由Go语言的`map[string]struct{}`结构直接序列化为JSON格式的，形式为每个键对应着 **值为空对象{}** 的JSON对象。如下面的例子所示：
```
{
    "8080": {},
    "53/udp": {},
    "2356/tcp": {}
}
```
其中的键可以是下面的格式：
+ `"port/tcp"`
+ `"port/udp"`
+ `"port"`
如果没有给出协议，默认使用`tcp`协议。这是创建容器使用的默认值，可以与命令行提供的端口列表合并。

#### Env 
`array of strings`
Env的每项都是 `VARNAME="var value"` 的格式。这是创建容器使用的默认值，可以与命令行提供的环境变量列表合并。

#### Entrypoint 
`array of strings`
容器启动时执行的命令参数列表。这是创建容器使用的默认值，可以被命令行提供的入口命令替换。

#### Cmd 
`array of strings`
容器启动时执行的命令参数列表（附加在`Entrypoint`之后）。这是创建容器使用的默认值，可以被命令行提供的入口命令替换。如果没有给出`Entrypoint`，那么`Cmd`列表的第一项将被认为是可执行的程序名。

#### Healthcheck 
`struct`
用以检查容器是否正常的测试命令，如下面的例子所示：
```
{
  "Test": [
      "CMD-SHELL",
      "/usr/bin/check-health localhost"
  ],
  "Interval": 30000000000,
  "Timeout":  10000000000,
  "Retries":  3
}
```

此结构有如下字段，

##### Test 
`array of strings`
用以检查容器是否正常的测试命令，可以是
+ `[]` : 继承父镜像的健康检查命令；
+ `["NONE"]` : 禁用健康检查；
+ `["CMD", arg1, arg2, ...]` : 直接执行命令和参数；
+ `["CMD-SHELL", command]` : 使用系统默认shell执行命令；

如果容器状态正常，测试命令退出后应返回 `0`，否则返回 `1`。
+ Interval `integer`：相邻两次尝试的间隔，单位为纳秒；
+ Timeout `integer`：认为异常的超时间隔，单位为纳秒；
+ Retries `integer`：认为异常的重试次数。

任何缺失的值都会从基础镜像继承。这是创建容器使用的默认值，可以与命令行提供的值合并（替代？）。

##### Volumes 
`struct`
创建容器时作为数据卷的一组目录。此JSON结构的特殊之处在于它是由Go语言的`map[string]struct{}`结构直接序列化为JSON格式的，形式为每个键对应着 **值为空对象{}** 的JSON对象。如下面的例子所示：
```
{
    "/var/my-app-data/": {},
    "/etc/some-config.d/": {},
}
```

##### WorkingDir 
`string`
容器入口程序的工作目录，这是创建容器使用的默认值，可以被命令行提供的值替代。

##### rootfs 
`struct`
rootfs结构是镜像各层的`DiffID`列表，此结构使镜像的描述文件的散列与各层的散列（及内容）相对应。rootfs有两个字段:
+ `type`，其值一般为 `layers`.
+ `diff_ids` 各层散列（`DiffID`）的数组，顺序为从最底层到最顶层。

下面是 rootfs 的一个例子：

```
"rootfs": {
  "diff_ids": [
    "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
    "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
    "sha256:13f53e08df5a220ab6d13c58b2bf83a59cbdc2e04d0a3f041ddf4b0ba4112d49"
  ],
  "type": "layers"
}
```

### history 
`struct`
`history`结构是描述每层历史的一组对象，顺序为从最底层到最顶层。每个对象有以下字段。
+ `created`: 创建的日期和时间，[ISO-8601格式](https://zh.wikipedia.org/wiki/ISO_8601)；
+ `author`: 创建的作者；
+ `created_by`: 创建该层的命令；
+ `comment`: 创建该层的注释；
+ `empty_layer`: 标识此项历史记录是否会创建一个文件变更集。如果值为`true`，则此项历史不会对应一个实际的文件集（如`ENV`命令就对层的文件没有影响）。
下面是 history 结构的一个例子：
```
"history": [
  {
    "created": "2015-10-31T22:22:54.690851953Z",
    "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
  },
  {
    "created": "2015-10-31T22:22:55.613815829Z",
    "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
    "empty_layer": true
  }
]
```

镜像的JSON文件中任何额外的字段应被认为是特定于实现的，如果无法处理，应该将其忽略。

## 创建镜像文件变更集
创建镜像文件变更集的例子如下：
首先，镜像的基础文件是一个空的目录，使用了随机生成的目录名`c3167915dc9d`（层的DiffID是基于目录内的文件内容生成的）。

然后在其中创建文件和目录:
```
c3167915dc9d/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

将目录`c3167915dc9d`提交为一个 `tar` 包（无压缩），其中包含如下的文件：

```
etc/my-app-config
bin/my-app-binary
bin/my-app-tools
```

如果要在此基础上更改文件，则创建一个新的目录，假如为`f60c56784b83`，将其初始化为父镜像的快照，即与目录`c3167915dc9d`的内容相同。
>注意：支持 Copy-on-Write 或联合机制的文件系统创建快照很高效。

```
f60c56784b83/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

然后添加一个配置目录`/etc/my-app.d`，其中包含默认的配置文件。可执行程序`my-app-tools`也更新了，以便处理新的配置文件路径。
修改后的目录`f60c56784b83`如下所示：
```
f60c56784b83/
    etc/
        my-app.d/
            default.cfg
    bin/
        my-app-binary
        my-app-tools
```

其中移除了`/etc/my-app-config`，然后创建了新的目录和文件`/etc/my-app.d/default.cfg`。`/bin/my-app-tools`也替换成新的版本。在将此目录提交为变更集之前，首先需要与其父镜像的快照`f60c56784b83`比较，找出增加、修改及删除的文件和目录。本例中找到如下的变更集：
```
增加：  /etc/my-app.d/default.cfg
修改：  /bin/my-app-tools
删除：  /etc/my-app-config
```

创建一个 **仅包含** 此变更集的tar包：增加和修改的文件内容及目录被完整地包含在tar包中；而删除的项则对应为相同路径的空文件，其文件名或目录名增加`.wh.`前缀（表示已删除）。
> 注意：无法直接创建以名称`.wh.`开头的文件或目录。

目录`f60c56784b83`生成的`tar` 包中有如下的文件：

```
/etc/my-app.d/default.cfg
/bin/my-app-tools
/etc/.wh.my-app-config
```

任何镜像都是由若干类似的文件变更集的tar包组成的。

## 镜像的组合格式
包含镜像完整内容的单一tar包格式如下：
- 镜像名：tag
- 镜像的 JSON 配置文件
- 各层的tar包

如镜像`library/busybox`的组合tar包内容如下（使用`tree`命令输出）：

```
.
├── 47bcc53f74dc94b1920f0b34f6036096526296767650f223433fe65c35f149eb.json
├── 5f29f704785248ddb9d06b90a11b5ea36c534865e9035e4022bb2e71d4ecbb9a
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── a65da33792c5187473faa80fa3e1b975acba06712852d1dea860692ccddf3198
│   ├── VERSION
│   ├── json
│   └── layer.tar
├── manifest.json
└── repositories
```

镜像的每层都对应一个目录，其名称是64个十六进制的字符，是根据该层的文件内容确定性地生成的。
> 注意：该目录名 **不必** 是层的`DiffID`或`ChainID`。

每个目录包含3个文件：
+ `VERSION` - `json`文件模式的版本号；
+ `json` - 旧的JSON格式镜像层元数据。v1.2版的镜像规范中，各层没有JSON元数据，但在v1版中是存在的。此文件是为了向后兼容v1版格式。
+ `layer.tar` - 该层的tar包。

注意：这个目录结构仅用于向后兼容。当前的实现使用`manifest.json`文件中列出的目录。

`VERSION`文件只是JSON元数据模式的版本号：`1.0`。

`repositories`也是一个JSON文件，包含镜像名和tag列表：
```
{  
    "busybox":{  
        "latest":"5f29f704785248ddb9d06b90a11b5ea36c534865e9035e4022bb2e71d4ecbb9a"
    }
}
```

其中有镜像的`repository`和一组tag列表。每个tag关联着镜像的 ID。该文件仅用于向后兼容。当前的实现使用`manifest.json`文件。

`manifest.json`文件是顶层镜像的JSON配置。
该文件包含以下元数据：
```
[
  {
    "Config": "47bcc53f74dc94b1920f0b34f6036096526296767650f223433fe65c35f149eb.json",
    "RepoTags": ["busybox:latest"],
    "Layers": [
      "a65da33792c5187473faa80fa3e1b975acba06712852d1dea860692ccddf3198/layer.tar",
      "5f29f704785248ddb9d06b90a11b5ea36c534865e9035e4022bb2e71d4ecbb9a/layer.tar"
    ]
  }
]
```

上面这个JSON数组中，每项都对应着一个镜像。
+ `Config` 指向该镜像的JSON文件；
+ `RepoTags` 是该镜像的名称；
+ `Layers` 指向镜像各层的 tar 包；
+ `Parent` 可选，指向其父镜像的 imageID，父镜像的ID必须在同一个 `manifest.json` 文件中存在。


不要把 `manifest.json` 与用来push和pull镜像的分发清单（distribution manifest）相混淆。
一般来说，支持v1.2版本镜像规范的实现将使用`manifest.json`文件，早期的实现仍使用 `*/json`和`repositories`文件。

# TODO
其它相关文档：
+ [Open Containers Initiative Image Spec](https://github.com/opencontainers/image-spec)
+ [Image Manifest Version 2, Schema 2](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md)
+ [Docker Registry HTTP API V2](https://github.com/docker/distribution/blob/master/docs/spec/api.md)
+ [Supporting Container Images in Mesos Containerizer](https://github.com/apache/mesos/blob/master/docs/container-image.md)

# 补充
>注意：不要把上面的 **镜像格式** 与镜像的 **主机存储布局** 搞混了。
+ 镜像格式是执行`docker save <镜像名或ID>`之后得到的对应镜像`tar`包的格式。
+ 镜像在主机的存储布局，以及镜像push和pull都 **不会** 用到打包成一个文件的镜像，因为这样不利于多个层并行加速下载和利用本地缓存的层。

>镜像的各层存在顺序依赖，而镜像也有父子继承关系。
最初Docker只支持AUFS存储驱动，但AUFS没有合并到Linux内核，虽然Ubuntu内置了AUFS，但RHEL/CentOS则需要添加对应的内核模块。目前Docker在RHEL/CentOS默认使用OverlayFS作为存储驱动。OverlayFS只支持上下两层，所以其主机存储布局与AUFS不同，但镜像格式不受影响。

## alpine镜像的主机存储布局
alpine镜像仅有一个层，比较简单。Ubuntu使用AUFS存储驱动的文件布局如下。layer.tar包 已经被解开了。
+ ./aufs是解开layer.tar后的文件内容；
+ ./aufs/mnt是容器文件系统的挂载点；
+ ./containers是创建的容器的读写层；
+ ./image/aufs/distribution中两个文件夹相当于正反查找的指针；
+ ./image/aufs/imagedb/content/sha256/[id] 接近镜像的JSON描述文件，但与上面的v1.2规范不完全一致；

```
# tree /var/lib/docker
/var/lib/docker
├── aufs
│   ├── diff
│   │   └── 0b9c9a223af5f795049b86fc4f3dace61a44ced8a08a3cc8ccad0699eecec951
│   │       ├── bin
│   │       │   ├── ash -> /bin/busybox
│   │       │  # ... alpine镜像中的文件列表（大部分是指向/bin/busybox的软链接）
│   ├── layers
│   │   └── 0b9c9a223af5f795049b86fc4f3dace61a44ced8a08a3cc8ccad0699eecec951
│   └── mnt
│       └── 0b9c9a223af5f795049b86fc4f3dace61a44ced8a08a3cc8ccad0699eecec951
├── containers
├── image
│   └── aufs
│       ├── distribution
│       │   ├── diffid-by-digest
│       │   │   └── sha256
│       │   │       └── cfc728c1c5584d8e0ae69368fc9c34d54d72651355573ba42554c2469a0a6299
│       │   └── v2metadata-by-diffid
│       │       └── sha256
│       │           └── e154057080f406372ebecadc0bfb5ff8a7982a0d13823bab1be5b86926c6f860
│       ├── imagedb
│       │   ├── content
│       │   │   └── sha256
│       │   │       └── 02674b9cb179d57c68b526733adf38b458bd31ba0abff0c2bf5ceca5bad72cd9
│       │   └── metadata
│       │       └── sha256
│       ├── layerdb
│       │   ├── sha256
│       │   │   └── e154057080f406372ebecadc0bfb5ff8a7982a0d13823bab1be5b86926c6f860
│       │   │       ├── cache-id 
│       │   │       ├── diff
│       │   │       ├── size
│       │   │       └── tar-split.json.gz # 如果是中间层，此处会有 parent 文件
│       │   └── tmp
│       └── repositories.json
├── network
│   └── files
│       └── local-kv.db
├── plugins
│   ├── storage
│   │   └── blobs
│   │       └── tmp
│   └── tmp
├── swarm
├── tmp
├── trust
└── volumes
    └── metadata.db

```