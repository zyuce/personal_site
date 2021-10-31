---
title: "[论文学习]Large-scale cluster management at Google with Borg"
date: 2021-09-10T17:18:50+08:00
draft: true
---

# 2.1 Workload
1.面向终端用户的、长期运行的、处理短暂的延时敏感的请求的服务被称为prod jobs，有较高的优先级

2.批量作业，运行几分钟或几天后完成，对于短暂的性能波动不敏感，被称为non-prod jobs

# 2.2 Clusters and cells
1.一个cell中的机器属于一个cluster，由连接他们的高性能数据中心规模的网络结构来定义， 一个cluster属于一个数据中心，一组数据中心组成一个site。一个cluster通常管理一个巨大的cell和一些小规模的测试、或特殊目的的cell。cell包含的机器数的中位数是10000机器。



# 2.3 Jobs and tasks
1.job包含一组task， job通过constraint定义task运行需要的机器属性，constraint包括hard和soft。task对应了运行在一个容器中的一组linux进程。task包含对资源需求的描述， 每个task通常和job中的其他task属性相同，但也可以被特殊覆盖。

2.borg程序是静态链接的， 以减少对运行环境的依赖。程序被borg构建成由二进制和数据组成的packages

3.job、task生命周期：





# 2.5 Priority, quota, and admission control
1.borg定义了不同的优先级种类（band），如monitoring, production, batch, best effort(也叫testing或free)

本文的prod job属于monitoring或production

2.preemption cascade（抢占瀑布）一个高优作业抢占一个低优作业，导致该低优作业又去下一个更低优作业。borg不允许同一band优先级的作业互相抢占

3.quota被用来表示在某钟优先级上、一段时间内的一组资源量（CPU，RAM，disk等）

4.通过超卖quota给低优先级来解决用户超买的问题， 在zero优先级中每个用户都有无限的quota



# 2.6 Naming and monitoring
1.Borg提供了发现服务， 通过cell name，job name，task number组成了BNS name（Borg name service）。Borg用BNS name将task的hostname和端口写入Chubby，用来进行RPC请求。BNS name同时还组成了task的DNS名字，可以通过域名访问

2.Borg通过HTTP健康心跳检查来监控task， 当http异常时会去重启task

3.Sigma服务提供给用户一个可视化Web界面， 可以检查作业、cell、日志、执行历史、结果。

4.Borg将job提交，task事件存储至Dremel中，这是一个使用类SQL交互的只读的数据存储。



# 3.Borg architecture
1.一个Borg cell由一个Borgmaster和每个机器上的Borglet组成



# 3.1 Borgmaster
1.Borgmaster由两个进程组成，主进程和调度器。

主进程负责处理RPC请求，如创建job、提供数据查询，还负责机器状态管理，与Borglet交互，提供web UI作为Sigma备份

2.Borgmaster拥有5个replica，每个replica维护了一个cell的状态的内存拷贝，数据同样通过高可用、分布式、基于Paxos存储在本地磁盘，每个cell都有一个Paxos leader处理改变cell状态的各种操作，如提交作业、销毁task。

3.Borgmaster的状态数据通过快照和日志结合的方式存储在Paxos中



# 3.2 Scheduling
1.调度器按优先级由高到低扫面task，在同一优先级中以round-robin的方式扫面确保用户之间是公平的，同时也避免了大作业线头阻塞（head-of-line blocking）

2.调度算法分两个部分， 首先是可行性检查，即找出可以跑该task的机器， 其次是打分，选出其中一个机器。

3.worst-fit为机器负载留出空间， 但是会形成更多碎片不利于大作业调度

4.best-fit有利于调度大作业， 但是造成部分机器资源彻底被浪费， 而且如果用户提交的作业所需资源高于填写的quota，比如定义了较低CPU的离线作业，会伤害其他应用性能。

5.作业启动延时（从job提交到task运行的时间）被关注， 中位数大约是25s， 其中包安装占用了80%的时间，其中的瓶颈是将包写入本地磁盘



# 3.3 Borglet
1.Borglet负责启停task、重启错误task、通过系统内核管理本地资源、滚动调试日志、上报机器状态给Borgmaster和其他监控系统。

2.Borgmaster每几秒钟请求Borglet获取机器状态并发送未完成的请求，这使得Borgmaster控制了通信频率，避免了明确的流速控制机制，也预防了恢复风暴

3.为了性能扩展， 每个Borgmaster replica运行了一个无状态的分片与其中部分Borglet通信，分片在replica扩缩容时重新计算。

为了弹性，Borglet上报它的全部信息， 但是分片只汇聚压缩并报导那些变化的信息。

4.如果Borglet连续几次没有响应Borgmaster的请求，机器将被标识下线并且上面的task将被重新调度至其他机器，当与Borglet的通信恢复时Borgmaster会让它kill掉这些已被重新调度的task以避免重复。

5.Borglet即使与Borgmaster失联也会正常执行操作，以防Borgmaster的replica全部故障



# 3.4 Scalability
1.为了处理更大的cell， 调度器被拆分为一个独立的进程， 与其他Borgmaster replica并行运行， 一个调度器replica基于一个cell的缓存数据进行操作， 它反复从Borgmaster主节点获取信息、执行调度、向主节点发送调度结果。

这与Omega的乐观并发控制相似

2.一些针对可扩展性的优化：

①分数缓存：Borg缓存打分结果，直到机器或task改变，忽略一些导致缓存不可用的微小资源量的改变

②等价类：同一constraint和资源需求的task只做一次可行性检查和打分

③宽松的随机化：在一个大cell中计算可行性和分数开销巨大， 因此调度器随机采集机器进行可行性检查直到发现满足数量的机器去打分。这有点类似Sparrow的分批采样， 同时也处理优先级、抢占、异质性、包安装开销。



# 4.可用性
1.通过以下方式减少task被驱逐的影响：

①自动重新调度被驱逐的task

②通过把job的task分发至不同的机器域、机架域、电源域来减少关联错误

③限制task被中断的比率以及一个job的task在机器、系统升级时可以被减少的数量

④使用声明状态的表达式和幂等操作，让异常客户端无害的重复请求

⑤对将不可达机器上的task迁移进行速度限制，因为难以区分是大规模机器故障还是网络分裂

⑥避免重复的会导致task或机器故障的task-机器配对

⑦通过一个logsaver task将关键数据写入本地盘，用户可以设置保留时间，通常是几天

2.Borgmaster可用性为99.99%



# 5.利用率


5.1 Evaluation methodology
cell compaction: given a workload, we found out how small a cell it could be fifitted into by removing machines until the workload no longer fifitted, repeatedly re-packing the workload from scratch to ensure that we didn’t get hung up on an unlucky confifiguration.



# 5.2 Cell sharing
通过将作业放在专用环境和共享环境（在离线混部、多用户共用）中并使用相同的clock speed，比较CPU CPI的差异来判断

CPI在同一时间间隔内与两个测量值正相关，机器上总体CPU使用情况，机器上任务数量

总体来说共享利大于弊



# 5.3 Large cells
使用多个小cell相比一个大cell处理相同的workload需要更多机器



# 5.4 Fine-grained resource requests
讨论是否需要提供修正资源的容器或虚拟机，而不是用户随意填写quota



# 5.5 Resource reclamation
开拓用户申请的过量的quota将它们提供给能接受低质量资源的作业， 如batch jobs

1.作业刚开始提交时保留资源等于作业请求的quota，300s后缓慢下降至真实资源加上一定的安全容错值，如果使用超过那保留资源会迅速增加

2.如果保留资源预测错误，机器所需资源可能不足以支持当前所有作业的需要，即使这些作业使用的资源不超过它们设置的limit， 这种情况会去kill non-prod作业



# 6 Isolation 
50%的机器运行9个甚至更多task，90分位的机器大约有25个task和4500个线程。



# 6.2 Performance isolation

1.task分为LS task（延时敏感）和其他叫做batch task

2.资源分为可压缩和不可压缩，可压缩的包括CPU cycles和磁盘 I/O，回收这些资源不需要kill task， 不可压缩资源如内存，磁盘空间，回收资源需要kill task













8.Lessons and future work
8.1 Lessons learned: the bad
1.仅使用Job组织task约束有局限性

用户只能通过job name和更高级别的工具来组织服务的拓扑，但这会在rolling update和resize job时产生问题

相对的， k8s使用通过label实现的pod来进行组织



2.每台机器只有一个ip导致事情复杂

Borg需要支持调度端口， Borglet需要实现端口隔离

相对的，k8s可以给每个task和service提供单独的ip地址



3.为高级用户提供优化牺牲了普通用户体验



8.2 Lessons learned: the good
