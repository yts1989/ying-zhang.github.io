title: 【译文修订】使用Borg在Google管理大规模集群
category: cloud
date: 2017-10-31
tags:

---

发表于EuroSys 2015的 ***Large-scale cluster management at Google with Borg*** 详细介绍了Google的Borg资源管理器。已经有网友“难易（HardySimpson）” 翻译了此文，这里对其稍作修订。之前读过两三遍此文，每遍都感觉有新的体会，这次修订就是为了更仔细地读一遍。


<!--more-->

---

**Large-scale cl uster management at Google with Borg**

作者：Abhishek Vermay, Luis Pedrosaz, Madhukar Korupolu, David Oppenheimer, Eric Tune, John Wilkes

http://research.google.com/pubs/pub43438.html 或 直接 [下载PDF全文](/doc/EuroSys15_Borg.pdf)

译者：难易 http://my.oschina.net/HardySimpson

EuroSys’15, http://dx.doi.org/10.1145/2741948.2741964

# 摘要

谷歌的Borg系统群集管理器运行几十万个以上的jobs，来自几千个不同的应用，跨多个集群，每个集群有上万个机器。

它通过管理控制、高效的任务包装、超售、和进程级别性能隔离实现了高利用率。它支持高可用性应用程序与运行时功能，最大限度地减少故障恢复时间，减少相关故障概率的调度策略。Borg简化了用户生活，通过提供一个声明性的工作规范语言，名称服务集成，实时作业监控，和分析和模拟系统行为的工具。

我们将会展现Borg系统架构和特点，重要的设计决策，定量分析它的一些策略，和十年以来的运维经验和学到的东西。

# 1. 简介

集群管理系统我们内部叫Borg，它管理、调度、开始、重启和监控谷歌运行的应用程序的生命周期。本文介绍它是怎么做到这些的。

Borg提供了三个主要的好处：它（1）隐藏资源管理和故障处理细节，使其用户可以专注于应用开发；（2）高可靠性和高可用性的操作，并支持应用程序做到高可靠高可用；（3）让我们在跨数以万计的机器上有效运行。Borg不是第一个来解决这些问题的系统，但它是在这个规模，这种程度的弹性和完整性下运行的为数不多的几个系统之一。

本文围绕这些主题来编写，包括了我们在生产环境运行十年的一些功力。

![Fig. 1](/img/borg-fig-01.png)

# 2.用户视角

Borg的用户是谷歌开发人员和系统管理员(网站可靠性工程师 SRE)，他们运行谷歌应用与服务。用户以job的方式提交他们的工作给Borg，job由一个或多个task组成，每个task含有同样的二进制程序。一个job在一个Borg的Cell里面跑，一个Cell是包括了多台机器的单元。这一节主要讲用户视角下的Borg系统。

## 2.1 工作负载

Borg Cell主要运行两种异构的工作负载。第一种是长期的服务，应该“永远”运行下去，并处理短时间的敏感请求（几微秒到几百毫秒）。这种服务是面向终端用户的产品如Gmail、Google Docs、网页搜索，内部基础设施服务（例如，Bigtable）。第二种是批处理任务，需要几秒到几天来完成，对短期性能波动不敏感。在一个Cell上混合运行了这两种负载，取决于他们的主要租户（比如说，有些Cell就是专门用来跑密集的批处理任务的）。工作负载也随着时间会产生变化：批处理任务做完就好，终端用户服务的负载是以每天为周期的。Borg需要把这两种情况都处理好。

Borg有一个2011年5月的负载数据[80]，已经被广泛的分析了[68,26，27，57，1]。

最近几年，很多应用框架是搭建在Borg上的，包括我们内部的MapReduce[23]、flumejava[18]、Millwheel[3]、Pregel[59]。这中间的大部分都是有一个控制器，可以提交job。前2个框架类似于YARN的应用管理器[76]。我们的分布式存储系统，例如GFS[34]和他的后继者CFS、Bigtable[19]、Megastore[8]都是跑在Borg上的。

在这篇文章里面，我们把高优先级的Borg的jobs定义为生产(prod)，剩下的是非生产的(non-prod)。大多长期服务是prod的，大部分批处理任务是non-prod的。在一个典型的Cell里面，prod job分配了70%的CPU资源然后实际用了60%；分配了55%的内存资源然后实际用了85%。在$5.5会展示分配和实际值的差是很重要的。

## 2.2 集群和Cell

一个Cell里面的所有机器都属于单个集群，集群是由高性能的数据中心级别的光纤网络连接起来的。一个集群安装在数据中心的一座楼里面，n座楼合在一起成为一个site。一个集群通常包括一个大的Cell还有一些小的或测试性质的Cell。我们尽量避免任何单点故障。

在测试的Cell之外，我们中等大小的Cell大概包括10000台机器；一些Cell还要大很多。一个Cell中的机器在很多方面都是异构的：大小(CPU,RAM,disk,network)、处理器类型、性能以及外部IP地址或flash存储。Borg隔离了这些差异，让用户单纯的选择用哪个Cell来跑任务，分配资源、安装程序和其它依赖、监控系统的健康并在故障时重启。

(译者：Cell其实就是逻辑上的集群)

## 2.3 job和task

一个Borg的job的属性有：名字、拥有者和有多少个task。job可以有一些约束，来指定这个job跑在什么架构的处理器、操作系统版本、是否有外部IP。约束可以是硬的或者软的。一个job可以指定在另一个job跑完后再开始。一个job只在一个Cell里面跑。

每个task包括了一组linux进程，跑在一台机器的一个容器内[62]。大部分Borg的工作负载没有跑在虚拟机(VM)里面，因为我们不想付出虚拟化的代价。而且，Borg在设计的时候还没硬件虚拟化什么事儿哪。

task也有一些属性，包括资源用量，在job中的排序。大多task的属性和job的通用task属性是一样的，也可以被覆盖 —— 例如，提供task专用的命令行参数，包括CPU核、内存、磁盘空间、磁盘访问速度、TCP端口等等，这些都是可以分别设置并按照一个好的粒度提供。我们不提供固定的资源的单元。Borg程序都是静态编译的，这样在跑的环境下就没有依赖，这些程序都被打成一个包，包括二进制和数据文件，能被Borg安装起来。

用户通过RPC来操作Borg的job，大多是从命令行工具，或者从我们的监控系统($2.6)。大多job描述文件是用一种申明式配置文件BCL -- GCL[12]的一个变种，会产生一个protobuf文件[67]。BCL有一些自己的关键字。GCL提供了lambda表达式来允许计算，这样就能让应用在环境里面调整自己的配置。上万个BCL配置文件超过一千行长，系统中累计跑了了千万行BCL。Borg的job配置很类似于Aurora配置文件[6]。

![Fig. 2](/img/borg-fig-02.png)

图2展现了job的和task的状态机和生命周期。

用户可以在运行时改变一个job中的task的属性，通过推送一个新的job配置给Borg。这个新的配置命令Borg更新task的规格。这就像是跑一个轻量级的，非原子性的事务，而且可以在提交后轻易再改回来。更新是滚动式的，在更新中可以限制task重启的数量，如果有太多task停掉，操作可以终止。

一些task更新，例如更新二进制程序，需要task重启；另外一些例如修改资源需求和限制会导致这个机器不适合跑现有的task，需要停止task再重新调度到别的机器上；还有一些例如修改优先级是可以不用重启或者移动task的。

task需要能够接受Unix的SIGTERM信号，在他们被强制发送SIGKILL之前，这样就有时间去做清理、保存状态、结束现有请求执行、拒绝新请求。实际的notice的delay bound。实践中，80%的task能正常处理终止信号。

## 2.4 Allocs

Borg的alloc(allocation的缩写)是在单台机器上的一组保留的资源配额，用来让一个或更多的task跑；这些资源一直分配在那边，无论有没有被用。allocs可以被分配出来给未来的task，用来保持资源在停止一个task和重启这个task之间，用来聚集不同jobs的tasks到同一台机器上——例如一个web server实例和附加的，用于把serverURL日志发送到一个分布式文件系统的日志搜集实例。一个alloc的资源管理方式和一台机器上的资源管理方式是类似的；多个tasks在一个alloc上跑并共享资源。如果一个alloc必须被重新定位到其他的机器上，那么它的task也要跟着重新调度。

一个alloc set就像一个job：它是一组allocs保留了多台机器上的资源。一旦alloc set被创建，一个或多个jobs就可以被提交进去跑。简而言之，我们会用task来表示一个alloc或者一个top-level task(一个alloc之外的)，用job来表示一个job或者alloc set。

## 2.5 优先级、配额和管理控制

当有超量的工作负载在运行的时候会发生什么事情？我们的解决方案是优先级和配额。

所有job都有优先级，一个小的正整数。高优先级的task可以优先获取资源，即使后面被杀掉。Borg定义了不重叠的优先级段给不同任务用，包括(优先级降序)：监控、生产、批任务、高性能(测试或免费)。在这篇文章里面，prod的jobs是在监控和生产段。

虽然一个降级的task总会在cell的其他地方找到一席之地。降级瀑布也有可能会发生，就是一个task降下来之后，把下面运行的task再挤到别的机器上，如此往复。为了避免这种情况，我们禁止了prod级task互相排挤。合理粒度的优先级在其他场景下也很有用——MapReduce的master跑的优先级比worker高一点，来保证他们的可用性。

优先级是jobs的相对重要性，决定了jobs在一个cell里面是跑还是等(pending)。配额则是用来决定jobs是否运行被调度。配额就是一组资源(CPU, RAM, disk)的数量在一个指定的优先级、一个指定的时间段(月这个量级)。数量决定了这个用户的job可以用的最多资源(例子：20TB内存和prod优先级从现在到7月在xx cell内)。配额检查是管理控制的一部分，不是调度层的：配额不足的任务在提交的时候就会被拒绝。

高优先级的配额总是花费的比低优先级要多。prod级的配额是被限制为一个cell里面实际的资源量，所以用户提交了prod级的job的配额时，可以期待这个job一定会跑，去掉一些碎片外。即使这样，我们鼓励用户多买一点比自己需要多一点的配额，很多用户超买是因为他们的应用程序的用户数量增长后需要的配额就大了。对于超买，我们的应对方案是低优先级资源的超售：所有用户在0优先级都可以用无限的配额，虽然在实际运行中这种情况很难跑起来。一个低优先级的job在资源不足时会保持等(pending)状态。

配额分配在Borg系统之外，和我们的物理资源计划有关。这些资源计划在不同的数据中心产生不同的价格和配额。用户jobs只在有足够配额和足够优先级之后才能启动。配额的使用让Dominant Resource Fairness(DRF)[29, 35, 36, 66]不是那么必要了。

Borg有一个容量系统给一些特殊权限给某些用户，例如，允许管理员删除或修改cell里面的job，或者允许用户区访问特定的内核特性或者让Borg对自己的job不做资源估算($5.5)。

## 2.6 命名和监控

光是创建和部署task是不够的：一个服务的客户端和其他系统需要能找到它们，即使它换了个地方。为了搞定这一点，Borg创造了一个稳定的“Borg name Service”(BNS)名字给每个task，这个名字包括了cell名字，job名字，和task编号。Borg把task的主机名和端口写入到一个持久化高可用文件里，以BNS名为文件名，放在Chubby[14]上。这个文件被我们的RPC系统使用，用来发现task的终端地址。BNS名称也是task的DNS名的基础构成部分，所以，cc cell的ubar用户的jfoo job的第50个task的DNS名称会是50.jfoo.ubar.cc.borg.google.com。Borg同时还会把job的大小和task的健康信息写入到Chubby在任何情况改变时，这样负载均衡就能知道怎么去路由请求了。

几乎所有的Borg的task都会包含一个内置的HTTP服务，用来发布健康信息和几千个性能指标(例如RPC延时)。Borg监控这些健康检查URL，把其中响应超时的和error的task重启。其他数据也被监控工具追踪并在Dashboards上展示，当服务级别对象(SLO)出问题时就会报警。

用户可以使用一个名叫Sigma的web UI，用来检查他们所有的job状态，一个特殊的cell，或者深入到某个job的某个task的资源用率，详细日志，操作历史，和最终命运。我们的应用产生大量的日志，都会被自动的滚动来避免塞满硬盘，会在一个task结束后保留一小段时间用来debug。如果一个job没有被跑起来，Borg会提供一个为什么挂起的解释，指导用户怎么修改这个job的资源需求来符合目前这个cell的情况。我们发布资源的使用方针，按照这个方针来做就容易被调度起来。

Borg记录所有的job提交和task时间，以及每task的资源使用细节在基础存储服务里面。这个存储服务有一个分布式的只读的SQL-like的交互式接口，通过Dremel[61]提供出来。这些数据在实时使用、debug、系统查错和长期容量规划上都很有用。这些数据也是Google集群负载追踪的数据来源之一[80].

所有这些特性帮助用户理解和debug Borg的行为和管理他们的job，并且帮助我们的SRE每个人管理超过上万台机器。

# 3. Borg架构

一个Borg的Cell包括一堆机器，一个逻辑的中心控制服务叫做Borgmaster，和在每台机器上跑的Borglet的agent进程(见图1)。所有Borg的组件都是用C++写的。

![Fig. 1](/img/borg-fig-01.png)

## 3.1 Borgmaster

Cell的Borgmaster由2个进程组成，主的Borgmaster进程和一个单独的scheduler($3.2)。主的Borgmaster处理所有客户端的RPC请求，例如修改状态(创建job)，提供数据读取服务(查找job)。它同时管理系统中所有组件(机器、task、allocs等等)的状态机，和Borglet通信，并且提供一个Sigma的备份Web UI。

Borgmaster在逻辑上是一个单进程，但实际上开了5个副本。每个副本维护了一个内存级别的cell状态拷贝，这些状态同时被记录在一个高可用、分布式、Paxos-based存储[55]放在这些副本的本地硬盘上。在一个cell里面，一个单独的被选举出来的master同时用于Paxos leader和状态修改器，用来处理所有改变cell状态的请求，例如提交一个job或者在一个机器上终止一个task。当cell启动或前一个master挂了时，Paxos算法会选举出一个master；这需要一个Chubby锁然后其他系统可以找到master。选举一个master或者换一个新的需要的典型事件是10s，但需要大概1分钟才能让一个大的cell内生效，因为一些内存状态要重构。当一个副本从网络隔离中恢复时，需要动态的从其他Paxos副本中重新同步自己的状态。

某个时刻的Borgmaster的状态被称为checkpoint，会被以快照形式+change log形式保存在Paxos存储里面。checkpoints有很多用途，包括把Borgmaster的状态恢复到以前的任意时刻(例如在处理一个请求之前，用来解决软件缺陷)；极端情况下手动修改checkpoints，形成一个持续的事件日志供今后用；或用于线下的在线仿真。

一个高仿真的Borgmaster叫Fauxmaster，可以用来读取checkpoint文件，包括一份完整的Borgmaster的代码，和Borglet的存根接口。它接受RPC来改变状态机和执行操作，例如调度所有阻塞的tasks，我们用它来debug错误，和它交互就和Borgmaster交互是一样的，同样我们也有一个仿真的Borglet可以用checkpoint重放真实的交互。用户可以单步调试看到系统中的所有过去的改变。Fauxmaster在这种情况下也很有用：多个这个类型的job比较合适？以及在改变cell配置前做一个安全检查(这个操作会把任何关键jobs给踢掉吗？)

## 3.2 调度 schedule

当一个job被提交的时候，Borgmaster会把它持久化的存储在Paxos存储上，然后把这个job的task放到等待(pending)的队列里面去。这个队列会被scheduler异步的扫描，然后分发task到有充足资源的机器上。scheduler主要是处理task的，不是job。扫描从高优先级到低优先级，在同个优先级上用round-robin的方式处理，以保证用户之间的公平性和避免头上的大job阻塞住。调度算法有2个部分：可行性检查(feasibility checking)，找到一台能跑task的机器，和打分(scoring)，找个一个最合适的机器。

在可行性检查这个阶段，scheduler会找到一组机器，都满足task的约束而且有足够可用的资源 —— 包括了一些已经分配给低优先级任务的可以被腾出来的资源。在打分阶段，scheduler会找到其中“最好”的机器。这个分数包括了用户的偏好，但主要是被内置的标准：例如最小化的倒腾其他task，找到已经有这个task安装包的，在电力和出错的可用域之间尽可能分散的，在单台机器上混合高低优先级的task以保证高峰期扩容的。

Borg原来用E-PVM[4]的变种算法来打分，在异构的资源上生成一个单一的分数，在调度一个task时最小化系统的改变。但在实践中，E-PVM最后把负载平均分配到所有机器，把扩展空间留给高峰期 —— 但这么做的代价是增加了碎片，尤其是对于大的task需要大部分机器的时候；我们有时候给这种分配取绰号叫“最差匹配”。

分配策略光谱的另一端是“最佳匹配”，把机器塞任务塞的越紧越好。这样就能留下一些空的机器给用户jobs(他们也跑存储服务)，所以处理大task就比较直接了，不过，紧分配会惩罚那些对自己所需资源预估不足的用户。这种策略会伤害爆发负载的应用，而且对需要低CPU的批处理任务特别不友好，这些任务可以被轻易调度到不用的资源上：20%的non-prod task需要小于0.1核的CPU。

我们目前的打分模型是一个混合的，试图减少搁浅的资源 —— 一些因为这台机器上资源没被全部用掉而剩下的。比起“最佳匹配”，这个模型提供了3%-5%的打包效率提升(在[78]里面定义的)

如果一台机器在打分后没有足够的资源运行新的task，Borg会驱逐(preempts)低优先级的任务，从最低优先级往上踢，直到资源够用。我们把被踢掉的task放到scheduler的等待(pending)队列里面去，而不是迁移或冬眠这些task。

task启动延迟(从job提交到task运行之间的时间段)是被我们持续关注的。这个时间差别很大，一般来说是25s。包安装耗费了这里面80%的时间：一个已知的瓶颈就是对本地硬盘的争抢。为了减少task启动时间，scheduler希望机器上已经有足够的包(程序和数据)：大部分包是只读的所以可以被分享和缓存。这是唯一一种Borg scheduler支持的数据本地化方式。顺便说一下，Borg分发包到机器的办法是树形的和BT类型的协议。

另外，scheduler用了某些技术来扩散到几万台机器的cell里面。($3.4)

## 3.3 Borglet

Borglet是部署在cell的每台机器上的本地Borg代理程序。它启动停止task；如果task失败就重启；通过修改OS内核设置来管理本地资源；滚动debug日志；把本机的状态上报给Borgmaster和其他监控系统。

Borgmaster每过几秒就会轮询所有的Borglet来获取机器当前的状态还有发送任何请求。这让Borgmaster能控制交流频率，避免一个显式的流控机制，而且防止了恢复风暴[9].

选举出来的master负责发送消息给Borglet并且根据响应更新cell的状态。为了性能可扩展，每个Borgmaster副本会运行一个无状态的连接分配(link shard)来处理和特定几个Borglet的交流；这个分配会在Borgmaster选举的时候重新计算。为了保证弹性，Borglet把所有状态都报上来，但是link shard会聚合和压缩这些信息到状态机，来减少选举出来的master的负载。

如果Borglet几次没有响应轮询请求，将会被标记为挂了(down)，然后上面跑的task会被重新分配到其他机器。如果通讯恢复，Borgmaster会让这个Borglet杀掉已经被分配出去的task，来避免重复。Borglet会继续常规的操作即使和Borgmaster恢复联系，所以目前跑的task和service保持运行以防所有的Borgmaster挂了。

## 3.4 可扩展性

我们还不知道Borg的可扩展性极限在哪里，每次我们碰到一个极限，我们就越过去。一个单独的Borgmaster可以管理一个cell里面几千台机器，若干个cell可以处理10000个任务每分钟。一个繁忙的Borgmaster使用10-14个CPU核以及50GB内存。我们用了几项技术来获得这种扩展性。

早期的Borgmaster有一个简单的，同步的循环来处理请求、调度tasks，和Borglet通信。为了处理大的cell，我们把scheduler分出来作为一个单独的进程，然后就可以和别的Borgmaster功能并行的跑，别的Borgmaster可以开副本来容错。一个scheduler副本操作一份cell的状态拷贝。它重复地：从选举出来的master获取状态改变(包括所有的分配的和pending的工作)；更新自己的本地拷贝，做调度工作来分配task；告诉选举出来的master这些分配。master会接受这些信息然后应用之，除非这些信息不适合(例如，过时了)，这些会在scheduler的下一个循环里面处理。这一切都符合Omega[69]的乐观并行策略精神，而且我们最近真的给Borg添加这种功能，对不同的工作负载用不同的scheduler来调度。

为了改进响应时间，我们增加了一些独立线程和Borglet通信、响应只读RPC。为了更高的性能，我们分享(分区)这些请求给5个Borgmaster副本$3.3。最后，这让99%的UI响应在1s以内，而95%的Borglet轮询在10s以内。

一些让Borg scheduler更加可扩展的东西：

分数缓存：评估一台机器的可用性和分数是比较昂贵的，所以Borg会一直缓存分数直到这个机器或者task变化了——例如，这台机器上的task结束了，一些属性修改了，或者task的需求改变了。忽略小的资源变化让缓存保质期变长。

同级别均化处理：同一个Borg job的task一般来说有相同的需求和资源，所以不用一个个等待的task每次都去找可用机器，这会把所有可用的机器打n次分。Borg会对相同级别的task找一遍可用机器打一次分。

适度随机：把一个大的Cell里面的所有机器都去衡量一遍可用性和打分是比较浪费的。所以scheduler会随机的检查机器，找到足够多的可用机器去打分，然后挑出最好的一个。这会减少task进入和离开系统时的打分次数和缓存失效。适度随机有点像Sparrow [65]的批处理采样技术，同样要面对优先级、驱逐、非同构系统和包安装的耗费。

在我们的实验中($5)，调度整个cell的工作负载要花几百秒，但不用上面几项技术的话会花3天以上的时间。一般来说，一个在线的调度从等待队列里面花半秒就能搞定。

# 4. 可用性

![Fig. 3](/img/borg-fig-03.png)

在大型分布式系统里面故障是很常见的[10,11,12]。图3展示了在15个cell里面task驱逐的原因。在Borg上跑的应用需要能处理这种事件，应用要支持开副本、存储数据到分布式存储这些技术，并能定期的做快照。即使这样，我们也尽可能的缓和这些事件造成的影响。例如，Borg：

自动的重新调度被驱逐的task，如果需要放到新机器上运行
通过把一个job分散到不同的可用域里面去，例如机器、机架、供电域
在机器、OS升级这些维护性工作时，降低在同一时刻的一个job中的task的关闭率
使用声明式的目标状态表示和幂等的状态改变做操作，这样故障的客户端可以无损的重新启动或安全的遗忘请求
对于失联的机器上的task，限制一定的比率去重新调度，因为很难去区分大规模的机器故障和网络分区
避免特定的会造成崩溃的task:机器的匹配
critical级别的中间数据写到本地硬盘的日志保存task很重要，就算这个task所属的alloc被终止或调度到其他机器上，也要恢复出来做。用户可以设置系统保持重复尝试多久：若干天是比较合理的做法。
一个关键的Borg设计特性是：就算Borgmaster或者Borglet挂了，task也会继续运行下去。不过，保持master运行也很重要，因为在它挂的时候新的jobs不能提交，或者结束的无法更新状态，故障的机器上的task也不能重新调度。

Borgmaster使用组合的技术在实践中保证99.99%的可用性：副本技术应对机器故障；管理控制应对超载；部署实例时用简单、底层的工具去减少外部依赖(译者：我猜测是rsync或者scp这种工具)。每个cell和其他cell都是独立的，这样减少了误操作关联和故障传染。为了达到这个目的，所以我们不搞大cell。

# 5. 使用效率

Borg的一个主要目的就是有效的利用Google的机器舰队，这可是一大笔财务投资：让效率提升几个百分点就能省下几百万美元。这一节讨论了和计算了一些Borg使用的技术和策略。

## 5.1 测度方法论

我们的job部署是有资源约束的，而且很少碰到负载高峰，我们的机器是异构的，我们从service job回收利用的资源跑batch job。所以，为了测量我们需要一个比“平均利用率”更抽象的标准。在做了一些实验后我们选择了cell密度(cell compaction)：给定一个负载，我们不断的从零开始(这样可以避免被一个倒霉的配置卡住)，部署到尽可能小的Cell里面去，直到再也不能从这个cell里面抽机器出来。这提供了一个清晰的终止条件，并促进了无陷阱的自动化比较，这里的陷阱指的是综合化的工作负载和建模[31]。一个定量的比较和估算技术可以看[78]，有不少微妙的细节。

我们不可能在线上的cell做性能实验，所以我们用了Fauxmaster来达到高保真的模拟效果，使用了真的在线cell的负载数据包括所有的约束、实际限制、保留和常用数据($5.5)。这些数据从2014-10-1 14:00 PDT的Borg快照(checkpoints)里面提取出来。(其他快照也产生类似的结论)。我们选取了15个Borg cell来出报告，先排除了特殊目的的、测试的、小的(<5000机器)的cell，然后从剩下的各种量级大小的cell中平均取样。

在压缩cell实验中为了保持机器异构性，我们随机选择去掉的机器。为了保持工作负载的异构性，我们保留了所有负载，除了那些对服务和存储需要有特定需求的。我们把那些需要超过一半cell的job的硬限制改成软的，允许不超过0.2%的task持续的pending如果它们过于挑剔机器；广泛的测试表明这些结果是可重复的。如果我们需要一个大的cell，就把原cell复制扩大；如果我们需要更多的cell，就复制几份cell。

所有的实验都每个cell重复11次，用不同的随机数发生器。在图上，我们用一个横线来表示最少和最多需要的机器，然后选择90%这个位置作为结果，平均或者居中的结论不会代表一个系统管理员会做的最优选择。我们相信cell压缩提供了一个公平一致的方式去比较调度策略：好的策略只需要更少的机器来跑相同的负载。

我们的实验聚焦在调度(打包)某个时间点的一个负载，而不是重放一段长期的工作踪迹。这部分是因为复制一个开放和关闭的队列模型比较困难，部分是因为传统的一段时间内跑完的指标和我们环境的长期跑服务不一样，部分是因为这样比较起来比较明确，部分是因为我们相信怎么整都差不多，部分是因为我们在消费20万个Borg CPU来做测试——即使在Google的量级，这也不是一个小数目(译者：就你丫理由多！)

![Fig. 4](/img/borg-fig-04.png)

在生产环境下，我们谨慎的留下了一些顶部空间给负载的增加，比如一些“黑天鹅”时间，负载高峰，机器故障，硬件升级，以及大范围故障(供电进灰)。图4显示了我们在现实世界中可以把cell压缩到多小。上面的基线是用来表示压缩大小的。

## 5.2 Cell的共享使用

几乎我们所有的机器都同时跑prod和non-prod的task：在共享Borg cell里有98%的机器同时跑这2种task，在所有Borg管理的机器里面有83%同时跑这2种task(我们有一些专用的Cell跑特定任务)。

![Fig. 5](/img/borg-fig-05.png)

鉴于很多其他的组织把面向用户应用和批处理应用在不同的集群上运行，我们设想一下如果我们也这么干会发生什么情况。图5展现了在一个中等大小的Cell上分开跑我们prod和non-prod的工作负载将需要20-30%多的机器。这是因为prod的job通常会保留一些资源来应对极少发生的负载高峰，但实际上在大多情况下不会用这些资源。Borg把这批资源回收利用了($5.5)来跑很多non-prod的工作，所以最终我们只需要更少的机器。

![Fig. 6](/img/borg-fig-06.png)

大部分Borg cell被几千个用户共享使用。图6展现了为什么。对这个测试，如果一个用户消费超过了10TiB内存(或100TiB)，我们就把这个用户的工作负载分离到一个单独的Cell里面去。我们目前的策略展现了它的威力：即使我们设置了这么高的阈值(来分离)，也需要2-16倍多的Cell，和20-150%多的机器。资源池的方案再次有效地节省了开销。

但是，或许把很多不相关的用户和job类型打包放到一台机器上，会造成CPU冲突，然后就需要更多的机器进行补偿？为了验证这一点，我们看一下在同一台机器，锁定时钟周期，每指令循环数CPI(cycles per instruction)在不同环境的task下是怎么变化的。在这种情况下，CPI是一个可比较的指标而且可以代表冲突度量，因为2倍的CPI意味着CPU密集型程序要跑2倍的时间。这些数据是从一周内12000个随机的prod的task中获取的，用硬件测量工具[83]取的，并且对采样做了权重，这样每秒CPU都是平等的。测试结果不是非常明显。

我们发现CPI在同一个时间段内和下面两个量正相关：这台机器上总的CPU使用量，以及(强相关)这个机器上同时跑的task数量；每往一台机器上增加1个task，就会增加0.3%的CPI(线性模型过滤数据)；增加一台10%的CPU使用率，就会增加小于2%的CPI。即使这已经是一个统计意义显著的正相关性，也只是解释了我们在CPI度量上看到的5%的变化，还有其他的因素支配着这个变化，例如应用程序固有的差别和特殊的干涉图案[24,83]。

比较我们从共享Cell和少数只跑几种应用的专用Cell获取的CPI采样，我们看到共享Cell里面的CPI平均值为1.58(σ=0.35,方差)，专用Cell的CPI平均值是1.53(σ=0.32,方差).也就是说，共享Cell的性能差3%。

为了搞定不同Cell的应用会有不同的工作负载，或者会有幸存者偏差(或许对冲突更敏感的程序会被挪到专用Cell里面去)，我们观察了Borglet的CPI，在所有Cell的所有机器上都会被运行。我们发现专用Cell的CPI平均值是1.20(σ=0.29,方差)，而共享Cell里面的CPI平均值为1.43(σ=0.45,方差)，暗示了在专用Cell上运行程序会比在共享Cell上快1.19倍，这就超过了CPU使用量轻负载的这个因素，轻微的有利于专用Cell。

这些实验确定了仓库级别的性能测试是比较微妙的，加强了[51]中的观察，并且得出了共享并没有显著的增加程序运行的开销。

不过，就算我们假设用了我们结果中最不好的数据，共享还是有益的：比起CPU的降速，在各个方案里面减少机器更重要，这会带来减少所有资源的开销，包括内存和硬盘，不仅仅是CPU。

## 5.3 大Cell

![Fig. 7](/img/borg-fig-07.png)

Google建立了大Cell，为了允许大的任务运行，也是为了降低资源碎片。我们通过把负载从一个cell分到多个小cell上来测试后面那个效应(降低碎片效应)，随机的把job用round-robin方式分配出去。图7展示了用很多小cell会明显的需要更多机器。

## 5.4 资源请求粒度

![Fig. 8](/img/borg-fig-08.png)

Borg用户请求的CPU单位是千分之一核，内存和硬盘单位是byte。(1核是一个CPU的超线程，在不同机器类型中的一个通用单位)。图8展现了这个粒度的好处：CPU核和内存只有少数的“最佳击球点”，以及这些资源很少的相关性。这个分布和[68]里面的基本差不多，除了我们看到大内存的请求在90%这个线上。

![Fig. 9](/img/borg-fig-09.png)

提供一个固定尺寸的容器和虚拟机，在IaaS(infrastructure-as-a-service)提供商里面或许是比较流行的，但不符合我们的需求。为了展现这一点，我们把CPU核和内存限制做成一个个尺寸，然后把prod的job按照大一点最近的尺寸去跑(取这2个维度的平方值之和最近，也就是2维图上的直线)，0.5核的CPU,1G的内存为差值。图9显示了一般情况下我们需要30-50%多的资源来运行。上限来自于把大的task跑在一整台机器上，这些task即使扩大四倍也没办法在原有Cell上压缩跑。下限是允许这些task等待(pending)。(这比[37]里面的数据要大100%，因为我们支持超过4中尺寸而且允许CPU和内存无限扩张)。

## 5.5 资源再利用

一个job可以声明一个限制资源，是每个task能强制保证的资源上限。Borg会先检查这个限制是不是在用户的配额内，然后检查具体的机器是否有那么多资源来调度这个task。有的用户会买超过他们需要的配额，也有用户会的task实际需要更多的资源去跑，因为Borg会杀掉那些需要更多的内存和硬盘空间的task，或者卡住CPU使用率不上去。另外，一些task偶尔需要使用他们的所有资源(例如，在一天的高峰期或者受到了一个拒绝服务攻击)，大多时候用不上那么多资源。

比起把那些分出来但不用的资源浪费掉，我们估计了一个task会用多少资源然后把其他的资源回收再利用给那些可以忍受低质量资源的工作，例如批处理job。这整个过程被叫做资源再利用(resource reclamation)。这个估值叫做task自留地资源(reservation)，被Borgmaster每过几秒就计算一次，是Borglet抓取的细粒度资源消费用率。最初的自留地资源被设置的和资源限制一样大；在300s之后，也就是启动那个阶段，自留地资源会缓慢的下降到实际用量加上一个安全值。自留地资源在实际用量超过它的时候会迅速上升。

Borg调度器(scheduler)使用限制资源来计算prod task的可用性($3.2)，所以这些task从来不依赖于回收的资源，也不提供超售的资源；对于non-prod的task，使用了目前运行task的自留地资源，这么新的task可以被调度到回收资源。

一台机器有可能因为自留地预估错度而导致运行时资源不足 —— 即使所有的task都在限制资源之内跑。如果这种情况发生了，我们杀掉或者限制non-prod task，从来不对prod task下手。

![Fig. 10](/img/borg-fig-10.png)

图10展示了如果没有资源再利用会需要更多的机器。在一个中等大小的Cell上大概有20%的工作负载跑在回收资源上。

![Fig. 11](/img/borg-fig-11.png)

图11可以看到更多的细节，包括回收资源、实际使用资源和限制资源的比例。一个超内存限制的task首先会被重新调度，不管优先级有多高，所以这样就很少有task会超过内存限制。另一方面，CPU使用率是可以轻易被卡住的，所以短期的超过自留地资源的高峰时没什么损害的。

图11暗示了资源再利用可能是没必要的保守：在自留地和实际使用中间有一大片差距。为了测试这一点，我们选择了一个生产cell然后调试它的预估参数到一个激进策略上，把安全区划小点，然后做了一个介于激进和基本之间的中庸策略跑，然后恢复到基本策略。

![Fig. 12](/img/borg-fig-12.png)

图12展现了结果。第二周自留地资源和实际资源的差值是最小的，比第三周要小，最大的是第一和第四周。和预期的一样，周2和周3的OOM率有一个轻微的提升。在复查了这个结果后，我们觉得利大于弊，于是把中庸策略的参数放到其他cell上部署运行。

# 6. 隔离性

50%的机器跑9个以上的task；最忙的10%的机器大概跑25个task，4500个线程[83]。虽然在应用间共享机器会增加使用率，也需要一个比较好的机制来保证task之间不互相冲突。包括安全和性能都不能互相冲突。

## 6.1 安全隔离

我们使用Linux chroot监狱作为同一台机器不同task之间主要的安全隔离机制。为了允许远程debug，我们以前会分发ssh key来自动给用户权限去访问跑他们task的机器，现在不这么干了。对大多数用户来说，现在提供的是borgssh命令，这个程序和Borglet协同，来构建一个ssh shell，这个shell和task运行在同样的chroot和cgroup下，这样限制就更加严格了。

VM和安全沙箱技术被使用在外部的软件上，在Google’s AppEngine (GAE) [38]和Google Compute Engine (GCE)环境下。我们把KVM进程中的每个hosted VM按照一个Borg task运行。

## 6.2 性能隔离

早期的Borglet使用了一种相对原始粗暴的资源隔离措施：事后内存、硬盘、CPU使用率检查，然后终止使用过多内存和硬盘的task，或者把用太多CPU的激进task通过Linux CPU优先级降下来。不过，很多粗暴的task还是很轻易的能影响同台机器上其他task的性能，然后很多用户就会多申请资源来让Borg减少调度的task数量，然后会导致系统资源利用率降低。资源回收可以弥补一些损失，但不是全部，因为要保证资源安全红线。在极端情况下，用户请求使用专用的机器或者cell。

目前，所有Borg task都跑在Linux cgroup-based资源容器[17,58,62]里面，Borglet操作这些容器的设置，这样就增强了控制因为操作系统内核在起作用。即使这样，偶尔还是有低级别的资源冲突(例如内存带宽和L3缓存污染)还是会发生，见[60,83]

为了搞定超负荷和超请求，Borg task有一个应用阶级(appclass)。最主要的区分在于延迟敏感latency-sensitive (LS)的应用和其他应用的区别，其他应用我们在文章里面叫batch。LS task是包括面向用户的应用和需要快速响应的共享基础设施。高优先级的LS task得到最高有待，可以为了这个把batch task一次饿个几秒种。

第二个区分在于可压缩资源(例如CPU循环，disk I/O带宽)都是速率性的可以被回收的，对于一个task可以降低这些资源的量而不去杀掉task；和不可压缩资源(例如内存、硬盘空间)这些一般来说不杀掉task就没法回收的。如果一个机器用光了不可压缩资源，Borglet马上就会杀掉task，从低优先级开始杀，直到剩下的自留地资源够用。如果机器用完了可压缩资源，Borglet会卡住使用率这样当短期高峰来到时不用杀掉任何task。如果情况没有改善，Borgmaster会从这个机器上去除一个或多个task。

Borglet的用户空间控制循环在未来预期的基础上给prod task分配内存，在内存压力基础上给non-prod task分配内存；从内核事件来处理Out-of-Memory (OOM)；杀掉那些想获取超过自身限制内存的task，或者在一个超负载的机器上实际超过负载时。Linux的积极文件缓存策略让我们的实现更负载一点，因为精确计算内存用量会麻烦很多。

为了增强性能隔离，LS task可以独占整个物理CPU核，不让别的LS task来用他们。batch task可以在任何核上面跑，不过他们只被分配了很少的和LS task共享的资源。Borglet动态的调整贪婪LS task的资源限制来保证他们不会把batch task饿上几分钟，有选择的在需要时使用CFS带宽控制[75]；光有共享是不行的，我们有多个优先级。

![Fig. 13](/img/borg-fig-13.png)

就像Leverich [56]，我们发现标准的Linux CPU调度(CFS)需要大幅调整来支持低延迟和高使用率。为了减少调度延迟，我们版本的CFS使用了额外的每cgroup历史[16]，允许LS task驱逐batch task，并且避免多个LS task跑在一个CPU上的调度量子效应(scheduling quantum，译者：或许指的是互相冲突？)。幸运的是，大多我们的应用使用的每个线程处理一个请求模型，这样就缓和了持久负载不均衡。我们节俭地使用cpusets来分配CPU核给有特殊延迟需求的应用。这些措施的一部分结果展现在图13里面。我们持续在这方面投入，增加了线程部署和CPU管理包括NUMA超线程、能源觉察(例如[81])，增加Borglet的控制精确度。

Task被允许在他们的限制范围内消费资源。其中大部分task甚至被允许去使用更多的可压缩资源例如CPU，充分利用没有被使用的资源。大概5%的LS task禁止这么做，主要是为了增加可预测性；小于1%的batch task也禁止。使用超量内存默认是被禁止的，因为这会增加task被杀的概率，不过即使这样，10%的LS task打开了这个限制，79%的batch task也开了因为这事MapReduce框架默认的。这事对资源再回收($5.5)的一个补偿。Batch task很乐意使用没有被用起来的内存，也乐意不时的释放一些可回收的内存：大多情况下这跑的很好，即使有时候batch task会被急需资源的LS task杀掉。

【编者的话】最后两章探讨的是相关工作和改进。从中可以看到从Borg到Kubernetes，他们也做了不少思考，而这方面的工作远远没有完善，一直在进行中。期待大家都能从Google的实践中学到一些东西，并分享出来。

# 7. 相关工作

资源调度在各个领域已经被研究了数十年了，包括在广域HPC超算集群中，在工作站网络中，在大规模服务器集群中。我们主要聚焦在最相关的大规模服务器集群这个领域。

最近的一些研究分析了集群趋势，来自于Yahoo、Google、和Facebook[20, 52, 63, 68, 70, 80, 82]，展现了这些现代的数据中心和工作负载在规模和异构化方面碰到的挑战。[69]包含了这些集群管理架构的分类。

Apache Mesos [45]把资源管理和应用部署做了分离，资源管理由中心管理器(类似于Bormaster+scheduler)和多种类的“框架”比如Hadoop [41]和Spark [73]，使用offer-based的机制。Borg则主要把这些几种在一起，使用request-based的机制，可以大规模扩展。DRF [29, 35, 36, 66]策略是内赋在Mesos里的；Borg则使用优先级和配额认证来替代。Mesos开发者已经宣布了他们的雄心壮志：推测性资源分配和回收，然后把[69]里面的问题都解决。

YARN [76]是一个Hadoop中心集群管理。每个应用都有一个管理器和中央资源管理器谈判；这和2008年开始Google MapReduce从Borg获取资源如出一辙。YARN的资源管理器最近才能容错。一个相关的开源项目是Hadoop Capacity Scheduler [42]，提供了多租户下的容量保证、多层队列、弹性共享和公平调度。YARN最近被扩展成支持多种资源类型、优先级、驱逐、和高级权限控制[21]。俄罗斯方块原型[40]支持了最大完工时间觉察的job打包。

Facebook的Tupperware [64]，是一个类Borg系统来调度cgroup容器；虽然只有少量资料泄露，看起来他也提供资源回收利用功能。Twitter有一个开源的Aurora[5]，一个类Borg的长进程调度器，跑在Mesos智商，有一个类似于Borg的配置语言和状态机。

来自于微软的Autopilot[48]提供了“自动化的软件部署和开通；系统监控，以及在软硬件故障时的修复操作”给微软集群。Borg生态系统提供了相同的特性，不过还有没说完的；Isaard [48]概括和很多我们想拥护的最佳实践。

Quincy[49]使用了一个网络流模型来提供公平性和数据局部性在几百个节点的DAG数据处理上。Borg用的是配额和优先级在上万台机器上把资源分配给用户。Quincy处理直接执行图在Borg之上。

Cosmos [44]聚焦在批处理上，重点在于用户获得对集群捐献的资源进行公平获取。它使用一个每job的管理器来获取资源；没有更多公开的细节。

微软的Apollo系统[13]使用了一个每job的调度器给短期存活的batch job使用，在和Borg差不多量级的集群下获取高流量输出。Apollo使用了一个低优先级后台任务随机执行策略来增加资源利用率，代价是有多天的延迟。Apollo几点提供一个预测矩阵，关于启动时间为两个资源维度的函数。然后调度器会综合计算启动开销、远程数据获取开销来决定部署到哪里，然后用一个随机延时来避免冲突。Borg用的是中央调度器来决定部署位置，给予优先级分配处理更多的资源维度，而且更关注高可用、长期跑的应用；Apollo也许可以处理更多的task请求并发。

阿里巴巴的Fuxi(译者：也就是伏羲啦) [84]支撑数据分析的负载，从2009年开始运行。就像Borgmaster，一个中央的FuxiMaster(也是做了高可用多副本)从节点上获取可用的资源信息、接受应用的资源请求，然后做匹配。伏羲增加了和Borg完全相反的调度策略：伏羲把最新的可用资源分配给队列里面请求的任务。就像Mesos，伏羲允许定义“虚拟资源”类型。只有系统的工作负载输出是公开的。

Omega [69]支持多并行，特别是“铅垂线”策略，粗略相当于Borgmaster加上它的持久存储和link shards(连接分配)。Omega调度器用的是乐观并行的方式去控制一个共享的cell观察和预期状态，把这些状态放在一个中央的存储里面，和Borglet用独立的连接器进行同步。Omega架构。Omage架构是被设计出来给多种不同的工作负载，这些工作负载都有自己的应用定义的RPC接口、状态机和调度策略(例如长期跑的服务端程序、多个框架下的batch job、存储基础设施、GCE上的虚拟机)。形成对比的是，Borg提供了一种“万灵药”，同样的RPC接口、状态机语义、调度策略，随着时间流逝规模和复杂度增加，需要支持更多的不同方式的负载，而可可扩展性目前来说还不算一个问题($3.4)

Google的开源Kubernetes系统[53]把应用放在Docker容器内[28]，分发到多机器上。它可以跑在物理机(和Borg一样)或跑在其他云比如GCE提供的主机上。Kubernetes的开发者和Borg是同一拨人而且正在狂开发中。Google提供了一个云主机版本叫Google Container Engine [39]。我们会在下一节里面讨论从Borg中学到了哪些东西用在了Kubernetes上。

在高性能计算社区有一些这个领域的长期传统工作(e.g., Maui, Moab, Platform LSF [2, 47, 50])；但是这和Google Cell所需要的规模、工作负载、容错性是完全不一样的。大概来说，这些系统通过让很多任务等待在一个长队列里面来获取极高的资源利用率。

虚拟化提供商例如VMware [77]和数据中心方案提供商例如HP and IBM [46]给了一个大概在1000台机器量级的集群解决方案。另外，一些研究小组用几种方式提升了资源调度质量(e.g., [25, 40, 72, 74])。

最后，就像我们所指出的，大规模集群管理的另外一个重要部分是自动化和无人化。[43]写了如何做故障计划、多租户、健康检查、权限控制、和重启动性来获得更大的机器数/操作员比。Borg的设计哲学也是这样的，让我们的一个SRE能支撑超过万台机器。

# 8. 经验教训和未来工作

在这一节中我们会聊一些十年以来我们在生产环境操作Borg得到的定性经验，然后描述下这些观察结果是怎么改善Kubernete[53]的设计。

## 8.1 教训

我们会从一些受到吐槽的Borg特性开始，然后说说Kubernetes是怎么干的。

**Jobs是唯一的task分组的机制。**Borg没有天然的方法去管理多个job组成单个实体，或者去指向相关的服务实例(例如，金丝雀和生产跟踪)。作为hack，用户把他们的服务拓扑编码写在job名字里面，然后用更高层的工具区解析这些名字。这个问题的另外一面是，没办法去指向服务的任意子集，这就导致了僵硬的语义，以至于无法滚动升级和改变job的实例数。

为了避免这些困难，Kubernetes不用job这个概念，而是用标签(label)来管理它的调度单位(pods)，标签是任意的键值对，用户可以把标签打在系统的所有对象上。这样，对于一个Borg job，就可以在pod上打上job:jobname这样的标签，其他的有用的分组也可以用标签来表示，例如服务、层级、发布类型(生产、测试、阶段)。Kubernetes用标签选择这种方式来选取对象，完成操作。这样就比固定的job分组更加灵活好用。

**一台机器一个IP把事情弄复杂了。**在Borg里面，所有一台机器上的task都使用同一个IP地址，然后共享端口空间。这就带来几个麻烦：Borg必须把端口当做资源来调度；task必须先声明他们需要多少端口，然后了解启动的时候哪些可以用；Borglet必须完成端口隔离；命名和RPC系统必须和IP一样处理端口。

非常感谢Linux namespace，虚拟机，IPv6和软件定义网络SDN。Kubernetes可以用一种更用户友好的方式来消解这些复杂性：所有pod和service都可以有一个自己的IP地址，允许开发者选择端口而不是委托基础设施来帮他们选择，这些就消除了基础设置管理端口的复杂性。

**给资深用户优化而忽略了初级用户。**Borg提供了一大堆针对“资深用户”的特性这样他们可以仔细的调试怎么跑他们的程序(BCL有230个参数的选项)：开始的目的是为了支持Google的大资源用户，提升他们的效率会带来更大的效益。但是很不幸的是这么复杂的API让初级用户用起来很复杂，约束了他们的进步。我们的解决方案是在Borg上又做了一些自动化的工具和服务，从实验中来决定合理的配置。这就让皮实的应用从实验中获得了自由：即使自动化出了麻烦的问题也不会导致灾难。

## 8.2 经验

另一方面，有不少Borg的设计是非常有益的，而且经历了时间考验。

**Allocs是有用的。**Borg alloc抽象导出了广泛使用的logsaver样式($2.4)和另一个流行样式：定期数据载入更新的web server。Allocs和packages允许这些辅助服务能被一个独立的小组开发。Kubernetes相对于alloc的设计是pod，是一个多个容器共享的资源封装，总是被调度到同一台机器上。Kubernetes用pod里面的辅助容器来替代alloc里面的task，不过思想是一样的。

**集群管理比task管理要做更多的事。**虽然Borg的主要角色是管理tasks和机器的生命周期，但Borg上的应用还是从其他的集群服务中收益良多，例如命名和负载均衡。Kubernetes用service抽象来支持命名和负载均衡：service有一个名字，用标签选择器来选择多个pod。在底下，Kubernetes自动的在这个service所拥有的pod之间自动负载均衡，然后在pod挂掉后被重新调度到其他机器上的时候也保持跟踪来做负载均衡。

**反观自省是至关重要的。**虽然Borg基本上是“just works”的，但当有出了问题后，找到这个问题的根源是非常有挑战性的。一个关键设计抉择是Borg把所有的debug信息暴露给用户而不是隐藏：Borg有几千个用户，所以“自助”是debug的第一步。虽然这会让我们很难抛弃一些用户依赖的内部策略，但这还是成功的，而且我们没有找到其他现实的替代方式。为了管理这么巨量的资源，我们提供了几层UI和debug工具，这样就可以升入研究基础设施本身和应用的错误日志和事件细节。

Kubernetes也希望重现很多Borg的自探查技术。例如它和cAdvisor [15] 一切发型用于资源监控，用Elasticsearch/Kibana [30] 和 Fluentd [32]来做日志聚合。从master可以获取一个对象的状态快照。Kubernetes有一个一致的所有组件都能用的事件记录机制(例如pod被调度、容器挂了)，这样客户端就能访问。

**master是分布式系统的核心.**Borgmaster原来被设计成一个单一的系统，但是后来，它变成了服务生态和用户job的核心。比方说，我们把调度器和主UI(Sigma)分离出来成为单独的进程，然后增加了权限控制、纵向横向扩展、重打包task、周期性job提交(cron)、工作流管理，系统操作存档用于离线查询。最后，这些让我们能够提升工作负载和特性集，而无需牺牲性能和可维护性。

Kubernetes的架构走的更远一些：它有一个API服务在核心，仅仅负责处理请求和维护底下的对象的状态。集群管理逻辑做成了一个小的、微服务类型的客户端程序和API服务通信，其中的副本管理器(replication controller)，维护在故障情况下pod的服务数量，还有节点管理器(node controller)，管理机器生命周期。

## 8.3 总结

在过去十年间所有几乎所有的Google集群负载都移到了Borg上。我们将会持续改进，并把学到的东西应用到Kubernetes上。

#鸣谢

这篇文章的作者同时也评审了这篇文章。但是几十个设计、实现、维护Borg组件和生态系统工程师才是这个系统成功的关键。我们在这里列表设计、实现、操作Borgmaster和Borglet的主要人员。如有遗漏抱歉。

Borgmaster主设计师和实现者有Jeremy Dion和Mark Vandevoorde，还有Ben Smith, Ken Ashcraft, Maricia Scott, Ming-Yee Iu, Monika Henzinger。Borglet的主要设计实现者是Paul Menage。

其他贡献者包括Abhishek Rai, Abhishek Verma, Andy Zheng, Ashwin Kumar, Beng-Hong Lim, Bin Zhang, Bolu Szewczyk, Brian Budge, Brian Grant, Brian Wickman, Chengdu Huang, Cynthia Wong, Daniel Smith, Dave Bort, David Oppenheimer, David Wall, Dawn Chen, Eric Haugen, Eric Tune, Ethan Solomita, Gaurav Dhiman, Geeta Chaudhry, Greg Roelofs, Grzegorz Czajkowski, James Eady, Jarek Kusmierek, Jaroslaw Przybylowicz, Jason Hickey, Javier Kohen, Jeremy Lau, Jerzy Szczepkowski, John Wilkes, Jonathan Wilson, Joso Eterovic, Jutta Degener, Kai Backman, Kamil Yurtsever, Kenji Kaneda, Kevan Miller, Kurt Steinkraus, Leo Landa, Liza Fireman, Madhukar Korupolu, Mark Logan, Markus Gutschke, Matt Sparks, Maya Haridasan, Michael Abd-El-Malek, Michael Kenniston, Mukesh Kumar, Nate Calvin, OnufryWojtaszczyk, Patrick Johnson, Pedro Valenzuela, PiotrWitusowski, Praveen Kallakuri, Rafal Sokolowski, Richard Gooch, Rishi Gosalia, Rob Radez, Robert Hagmann, Robert Jardine, Robert Kennedy, Rohit Jnagal, Roy Bryant, Rune Dahl, Scott Garriss, Scott Johnson, Sean Howarth, Sheena Madan, Smeeta Jalan, Stan Chesnutt, Temo Arobelidze, Tim Hockin, Todd Wang, Tomasz Blaszczyk, TomaszWozniak, Tomek Zielonka, Victor Marmol, Vish Kannan, Vrigo Gokhale, Walfredo Cirne, Walt Drummond, Weiran Liu, Xiaopan Zhang, Xiao Zhang, Ye Zhao, Zohaib Maya.

Borg SRE团队也是非常重要的，包括Adam Rogoyski, Alex Milivojevic, Anil Das, Cody Smith, Cooper Bethea, Folke Behrens, Matt Liggett, James Sanford, John Millikin, Matt Brown, Miki Habryn, Peter Dahl, Robert van Gent, Seppi Wilhelmi, Seth Hettich, Torsten Marek, and Viraj Alankar。Borg配置语言(BCL)和borgcfg工具是Marcel van Lohuizen, Robert Griesemer制作的。

谢谢我们的审稿人(尤其是especially Eric Brewer, Malte Schwarzkopf and Tom Rodeheffer)，以及我们的牧师Christos Kozyrakis，对这篇论文的反馈。

#参考文献

[1] O. A. Abdul-Rahman and K. Aida. Towards understanding the usage behavior of Google cloud users: the mice and elephants phenomenon. In Proc. IEEE Int’l Conf. on Cloud Computing Technology and Science (CloudCom), pages 272–277, Singapore, Dec. 2014.

[2] Adaptive Computing Enterprises Inc., Provo, UT. MauiScheduler Administrator’s Guide, 3.2 edition, 2011.

[3] T. Akidau, A. Balikov, K. Bekiro˘glu, S. Chernyak, J. Haberman, R. Lax, S. McVeety, D. Mills, P. Nordstrom,and S. Whittle. MillWheel: fault-tolerant stream processing at internet scale. In Proc. Int’l Conf. on Very Large Data Bases (VLDB), pages 734–746, Riva del Garda, Italy, Aug.2013.

[4] Y. Amir, B. Awerbuch, A. Barak, R. S. Borgstrom, and A. Keren. An opportunity cost approach for job assignment in a scalable computing cluster. IEEE Trans. Parallel Distrib.Syst., 11(7):760–768, July 2000.

[5] Apache Aurora.http://aurora.incubator.apache.org/, 2014.

[6] Aurora Configuration Tutorial. https://aurora.incubator.apache.org/documentation/latest/configuration-tutorial/,2014.

[7] AWS. Amazon Web Services VM Instances. http://aws.amazon.com/ec2/instance-types/, 2014.

[8] J. Baker, C. Bond, J. Corbett, J. Furman, A. Khorlin, J. Larson, J.-M. Leon, Y. Li, A. Lloyd, and V. Yushprakh. Megastore: Providing scalable, highly available storage for interactive services. In Proc. Conference on Innovative Data Systems Research (CIDR), pages 223–234, Asilomar, CA, USA, Jan. 2011.

[9] M. Baker and J. Ousterhout. Availability in the Sprite distributed file system. Operating Systems Review,25(2):95–98, Apr. 1991.

[10] L. A. Barroso, J. Clidaras, and U. H¨olzle. The datacenter as a computer: an introduction to the design of warehouse-scale machines. Morgan Claypool Publishers, 2nd edition, 2013.

[11] L. A. Barroso, J. Dean, and U. Holzle. Web search for a planet: the Google cluster architecture. In IEEE Micro, pages 22–28, 2003.

[12] I. Bokharouss. GCL Viewer: a study in improving the understanding of GCL programs. Technical report, Eindhoven Univ. of Technology, 2008. MS thesis.

[13] E. Boutin, J. Ekanayake, W. Lin, B. Shi, J. Zhou, Z. Qian, M. Wu, and L. Zhou. Apollo: scalable and coordinated scheduling for cloud-scale computing. In Proc. USENIX Symp. on Operating Systems Design and Implementation (OSDI), Oct. 2014.

[14] M. Burrows. The Chubby lock service for loosely-coupled distributed systems. In Proc. USENIX Symp. on Operating Systems Design and Implementation (OSDI), pages 335–350,Seattle, WA, USA, 2006.

[15] cAdvisor. https://github.com/google/cadvisor, 2014

[16] CFS per-entity load patches. http://lwn.net/Articles/531853, 2013.

[17] cgroups. http://en.wikipedia.org/wiki/Cgroups, 2014.

[18] C. Chambers, A. Raniwala, F. Perry, S. Adams, R. R. Henry, R. Bradshaw, and N. Weizenbaum. FlumeJava: easy, efficient data-parallel pipelines. In Proc. ACM SIGPLAN Conf. on Programming Language Design and Implementation (PLDI), pages 363–375, Toronto, Ontario, Canada, 2010.

[19] F. Chang, J. Dean, S. Ghemawat, W. C. Hsieh, D. A. Wallach, M. Burrows, T. Chandra, A. Fikes, and R. E. Gruber. Bigtable: a distributed storage system for structured data. ACM Trans. on Computer Systems, 26(2):4:1–4:26, June 2008.

[20] Y. Chen, S. Alspaugh, and R. H. Katz. Design insights for MapReduce from diverse production workloads. Technical Report UCB/EECS–2012–17, UC Berkeley, Jan. 2012.

[21] C. Curino, D. E. Difallah, C. Douglas, S. Krishnan, R. Ramakrishnan, and S. Rao. Reservation-based scheduling: if you’re late don’t blame us! In Proc. ACM Symp. on Cloud Computing (SoCC), pages 2:1–2:14, Seattle, WA, USA, 2014.

[22] J. Dean and L. A. Barroso. The tail at scale. Communications of the ACM, 56(2):74–80, Feb. 2012.

[23] J. Dean and S. Ghemawat. MapReduce: simplified data processing on large clusters. Communications of the ACM, 51(1):107–113, 2008.

[24] C. Delimitrou and C. Kozyrakis. Paragon: QoS-aware scheduling for heterogeneous datacenters. In Proc. Int’l Conf. on Architectural Support for Programming Languages and Operating Systems (ASPLOS), Mar. 201.

[25] C. Delimitrou and C. Kozyrakis. Quasar: resource-efficient and QoS-aware cluster management. In Proc. Int’l Conf. on Architectural Support for Programming Languages and Operating Systems (ASPLOS), pages 127–144, Salt Lake City, UT, USA, 2014.

[26] S. Di, D. Kondo, and W. Cirne. Characterization and comparison of cloud versus Grid workloads. In International Conference on Cluster Computing (IEEE CLUSTER), pages 230–238, Beijing, China, Sept. 2012.

[27] S. Di, D. Kondo, and C. Franck. Characterizing cloud applications on a Google data center. In Proc. Int’l Conf. on Parallel Processing (ICPP), Lyon, France, Oct. 2013.

[28] Docker Project. https://www.docker.io/, 2014.

[29] D. Dolev, D. G. Feitelson, J. Y. Halpern, R. Kupferman, and N. Linial. No justified complaints: on fair sharing of multiple resources. In Proc. Innovations in Theoretical Computer Science (ITCS), pages 68–75, Cambridge, MA, USA, 2012.

[30] ElasticSearch. http://www.elasticsearch.org, 2014.

[31] D. G. Feitelson. Workload Modeling for Computer Systems Performance Evaluation. Cambridge University Press, 2014.

[32] Fluentd. http://www.fluentd.org/, 2014.

[33] GCE. Google Compute Engine. http: //cloud.google.com/products/compute-engine/, 2014.

[34] S. Ghemawat, H. Gobioff, and S.-T. Leung. The Google File System. In Proc. ACM Symp. on Operating Systems Principles (SOSP), pages 29–43, Bolton Landing, NY, USA, 2003. ACM.

[35] A. Ghodsi, M. Zaharia, B. Hindman, A. Konwinski, S. Shenker, and I. Stoica. Dominant Resource Fairness: fair allocation of multiple resource types. In Proc. USENIX Symp. on Networked Systems Design and Implementation (NSDI), pages 323–326, 2011.

[36] A. Ghodsi, M. Zaharia, S. Shenker, and I. Stoica. Choosy: max-min fair sharing for datacenter jobs with constraints. In Proc. European Conf. on Computer Systems (EuroSys), pages 365–378, Prague, Czech Republic, 2013.

[37] D. Gmach, J. Rolia, and L. Cherkasova. Selling T-shirts and time shares in the cloud. In Proc. IEEE/ACM Int’l Symp. on Cluster, Cloud and Grid Computing (CCGrid), pages 539–546, Ottawa, Canada, 2012.

[38] Google App Engine. http://cloud.google.com/AppEngine, 2014.

[39] Google Container Engine (GKE). https://cloud.google.com/container-engine/, 2015.

[40] R. Grandl, G. Ananthanarayanan, S. Kandula, S. Rao, and A. Akella. Multi-resource packing for cluster schedulers. In Proc. ACM SIGCOMM, Aug. 2014.

[41] Apache Hadoop Project. http://hadoop.apache.org/, 2009.

[42] Hadoop MapReduce Next Generation – Capacity Scheduler. http: //hadoop.apache.org/docs/r2.2.0/hadoop-yarn/ hadoop-yarn-site/CapacityScheduler.html, 2013.

[43] J. Hamilton. On designing and deploying internet-scale services. In Proc. Large Installation System Administration Conf. (LISA), pages 231–242, Dallas, TX, USA, Nov. 2007.

[44] P. Helland. Cosmos: big data and big challenges. http://research.microsoft.com/en-us/events/ fs2011/helland_cosmos_big_data_and_big\ _challenges.pdf, 2011.

[45] B. Hindman, A. Konwinski, M. Zaharia, A. Ghodsi, A. Joseph, R. Katz, S. Shenker, and I. Stoica. Mesos: a platform for fine-grained resource sharing in the data center. In Proc. USENIX Symp. on Networked Systems Design and Implementation (NSDI), 2011.

[46] IBM Platform Computing. http://www-03.ibm.com/ systems/technicalcomputing/platformcomputing/ products/clustermanager/index.html.

[47] S. Iqbal, R. Gupta, and Y.-C. Fang. Planning considerations for job scheduling in HPC clusters. Dell Power Solutions, Feb. 2005.

[48] M. Isaard. Autopilot: Automatic data center management. ACM SIGOPS Operating Systems Review, 41(2), 2007.

[49] M. Isard, V. Prabhakaran, J. Currey, U. Wieder, K. Talwar, and A. Goldberg. Quincy: fair scheduling for distributed computing clusters. In Proc. ACM Symp. on Operating Systems Principles (SOSP), 2009.

[50] D. B. Jackson, Q. Snell, and M. J. Clement. Core algorithms of the Maui scheduler. In Proc. Int’l Workshop on Job Scheduling Strategies for Parallel Processing, pages 87–102. Springer-Verlag, 2001.

[51] M. Kambadur, T. Moseley, R. Hank, and M. A. Kim. Measuring interference between live datacenter applications. In Proc. Int’l Conf. for High Performance Computing, Networking, Storage and Analysis (SC), Salt Lake City, UT, Nov. 2012.

[52] S. Kavulya, J. Tan, R. Gandhi, and P. Narasimhan. An analysis of traces from a production MapReduce cluster. In Proc. IEEE/ACM Int’l Symp. on Cluster, Cloud and Grid Computing (CCGrid), pages 94–103, 2010.

[53] Kubernetes. http://kubernetes.io, Aug. 2014.

[54] Kernel Based Virtual Machine. http://www.linux-kvm.org.

[55] L. Lamport. The part-time parliament. ACM Trans. on Computer Systems, 16(2):133–169, May 1998.

[56] J. Leverich and C. Kozyrakis. Reconciling high server utilization and sub-millisecond quality-of-service. In Proc. European Conf. on Computer Systems (EuroSys), page 4, 2014.

[57] Z. Liu and S. Cho. Characterizing machines and workloads on a Google cluster. In Proc. Int’l Workshop on Scheduling and Resource Management for Parallel and Distributed Systems (SRMPDS), Pittsburgh, PA, USA, Sept. 2012.

[58] Google LMCTFY project (let me contain that for you). http://github.com/google/lmctfy, 2014.

[59] G. Malewicz, M. H. Austern, A. J. Bik, J. C. Dehnert, I. Horn, N. Leiser, and G. Czajkowski. Pregel: a system for large-scale graph processing. In Proc. ACM SIGMOD Conference, pages 135–146, Indianapolis, IA, USA, 2010.

[60] J. Mars, L. Tang, R. Hundt, K. Skadron, and M. L. Soffa. Bubble-Up: increasing utilization in modern warehouse scale computers via sensible co-locations. In Proc. Int’l Symp. on Microarchitecture (Micro), Porto Alegre, Brazil, 2011.

[61] S. Melnik, A. Gubarev, J. J. Long, G. Romer, S. Shivakumar, M. Tolton, and T. Vassilakis. Dremel: interactive analysis of web-scale datasets. In Proc. Int’l Conf. on Very Large Data Bases (VLDB), pages 330–339, Singapore, Sept. 2010.

[62] P. Menage. Linux control groups. http://www.kernel. org/doc/Documentation/cgroups/cgroups.txt, 2007–2014.

[63] A. K. Mishra, J. L. Hellerstein, W. Cirne, and C. R. Das. Towards characterizing cloud backend workloads: insights from Google compute clusters. ACM SIGMETRICS Performance Evaluation Review, 37:34–41, Mar. 2010.

[64] A. Narayanan. Tupperware: containerized deployment at Facebook. http://www.slideshare.net/dotCloud/ tupperware-containerized-deployment-at-facebook, June 2014.

[65] K. Ousterhout, P. Wendell, M. Zaharia, and I. Stoica. Sparrow: distributed, low latency scheduling. In Proc. ACM Symp. on Operating Systems Principles (SOSP), pages 69–84, Farminton, PA, USA, 2013.

[66] D. C. Parkes, A. D. Procaccia, and N. Shah. Beyond Dominant Resource Fairness: extensions, limitations, and indivisibilities. In Proc. Electronic Commerce, pages 808–825, Valencia, Spain, 2012.

[67] Protocol buffers. https: //developers.google.com/protocol-buffers/, and https://github.com/google/protobuf/., 2014.

[68] C. Reiss, A. Tumanov, G. Ganger, R. Katz, and M. Kozuch. Heterogeneity and dynamicity of clouds at scale: Google trace analysis. In Proc. ACM Symp. on Cloud Computing (SoCC), San Jose, CA, USA, Oct. 2012.

[69] M. Schwarzkopf, A. Konwinski, M. Abd-El-Malek, and J. Wilkes. Omega: flexible, scalable schedulers for large compute clusters. In Proc. European Conf. on Computer Systems (EuroSys), Prague, Czech Republic, 2013.

[70] B. Sharma, V. Chudnovsky, J. L. Hellerstein, R. Rifaat, and C. R. Das. Modeling and synthesizing task placement constraints in Google compute clusters. In Proc. ACM Symp. on Cloud Computing (SoCC), pages 3:1–3:14, Cascais, Portugal, Oct. 2011.

[71] E. Shmueli and D. G. Feitelson. On simulation and design of parallel-systems schedulers: are we doing the right thing? IEEE Trans. on Parallel and Distributed Systems, 20(7):983–996, July 2009.

[72] A. Singh, M. Korupolu, and D. Mohapatra. Server-storage virtualization: integration and load balancing in data centers. In Proc. Int’l Conf. for High Performance Computing, Networking, Storage and Analysis (SC), pages 53:1–53:12, Austin, TX, USA, 2008.

[73] Apache Spark Project. http://spark.apache.org/, 2014.

[74] A. Tumanov, J. Cipar, M. A. Kozuch, and G. R. Ganger. Alsched: algebraic scheduling of mixed workloads in heterogeneous clouds. In Proc. ACM Symp. on Cloud Computing (SoCC), San Jose, CA, USA, Oct. 2012.

[75] P. Turner, B. Rao, and N. Rao. CPU bandwidth control for CFS. In Proc. Linux Symposium, pages 245–254, July 2010.

[76] V. K. Vavilapalli, A. C. Murthy, C. Douglas, S. Agarwal, M. Konar, R. Evans, T. Graves, J. Lowe, H. Shah, S. Seth, B. Saha, C. Curino, O. O’Malley, S. Radia, B. Reed, and E. Baldeschwieler. Apache Hadoop YARN: Yet Another Resource Negotiator. In Proc. ACM Symp. on Cloud Computing (SoCC), Santa Clara, CA, USA, 2013.

[77] VMware VCloud Suite. http://www.vmware.com/products/vcloud-suite/.

[78] A. Verma, M. Korupolu, and J. Wilkes. Evaluating job packing in warehouse-scale computing. In IEEE Cluster, pages 48–56, Madrid, Spain, Sept. 2014.

[79] W. Whitt. Open and closed models for networks of queues. AT&T Bell Labs Technical Journal, 63(9), Nov. 1984.

[80] J. Wilkes. More Google cluster data. http://googleresearch.blogspot.com/2011/11/ more-google-cluster-data.html, Nov. 2011.

[81] Y. Zhai, X. Zhang, S. Eranian, L. Tang, and J. Mars. HaPPy: Hyperthread-aware power profiling dynamically. In Proc. USENIX Annual Technical Conf. (USENIX ATC), pages 211–217, Philadelphia, PA, USA, June 2014. USENIX Association.

[82] Q. Zhang, J. Hellerstein, and R. Boutaba. Characterizing task usage shapes in Google’s compute clusters. In Proc. Int’l Workshop on Large-Scale Distributed Systems and Middleware (LADIS), 2011.

[83] X. Zhang, E. Tune, R. Hagmann, R. Jnagal, V. Gokhale, and J. Wilkes. CPI2: CPU performance isolation for shared compute clusters. In Proc. European Conf. on Computer Systems (EuroSys), Prague, Czech Republic, 2013.

[84] Z. Zhang, C. Li, Y. Tao, R. Yang, H. Tang, and J. Xu. Fuxi: a fault-tolerant resource management and job scheduling system at internet scale. In Proc. Int’l Conf. on Very Large Data Bases (VLDB), pages 1393–1404. VLDB Endowment Inc., Sept. 2014.

# 勘误 2015-04-23

自从胶片版定稿后，我们发现了若干疏忽和歧义。

## 用户视角

SRE干的比SA(system administration)要多得多：他们是Google生产服务的负责工程师。他们设计和实现软件，包括自动化系统、管理应用、底层基础设施服务来保证Google这个量级的高可靠和高性能。

## 鸣谢

我们不小心忽略了Brad Strand, Chris Colohan, Divyesh Shah, Eric Wilcox, and Pavanish Nirula。

## 参考文献

[1] Michael Litzkow, Miron Livny, and Matt Mutka. "Condor - A Hunter of Idle Workstations". In Proc. Int'l Conf. on Distributed Computing Systems (ICDCS) , pages 104-111, June 1988.

[2] Rajesh Raman, Miron Livny, and Marvin Solomon. "Matchmaking: Distributed Resource Management for High Throughput Computing". In Proc. Int'l Symp. on High Performance Distributed Computing (HPDC) , Chicago, IL, USA, July 1998.

