---
title: "离在线混部调研"
date: 2022-11-15T16:10:03+08:00
draft: false
tags: ["混部","调度", "资源隔离"]
categories: ["混部"]
keywords: 
  - 混部
  - 调度
  - 资源隔离
---


## 1. 背景

公司战略目标：降本增效

HPA上线使得在线业务的波谷期空闲出大量的计算资源

大数据业务云原生化是必然的趋势，公司大数据团队正在着手落地MR/Spark类的离线业务云原生化

## 2. 离在线混部的目标

1. 尽可以能的提高资源利用率
2. 保证在线业务的SLA
3. 保证离线业务运行良好

## 3. 业界离在线混部方案分类

按照混合类型分： 分时复用和部分混合和全部混合

按照隔离类型分： 资源独享和资源共享

按照部署方式分： 虚拟机部署和容器部署

这三个维度的组合，目前实际应用中主要是`分时复用 + 独占资源 + 物理机部署`、`部分混合 + 独占资源 + 容器部署`、`全部混合 + 共享资源 + 容器部署` 这三种模式，这三种模式其实也是离在线混部的三个阶段。

### 3.1 分时复用 + 独占资源 + 物理机部署

这种组合属于入门级的在离线混部选择，比如物理机运行服务且分时整机腾挪

好处是能够快速实现在离线混部，收获成本降低的红利。技术门槛上，这种方式规避了前面说的复杂的资源隔离问题，并且调度决策足够清晰，服务部署节奏有明确的预期，整个系统可以设计得比较简单。缺点是没有充分发挥出在离线混部的资源利用率潜力，目前主要是一些初创企业在应用。阿里早期在大促期间，将所有离线作业节点下线换上在线服务，也可以看做这种形态的近似版本

### 3.2 部分混合 + 独占资源 + 容器部署

在这种模型下，业务开发人员将服务部署在云原生部署平台，选择某些指标（大部分伴随着流量潮汐特性）来衡量服务负载，平台会按照业务指定规则对服务节点数量扩缩容。当流量低峰期来临时，在线服务会有大量碎片资源可释放，部署平台会整理碎片资源，将碎片资源化零为整后，以整机的方式将资源租借给离线作业使用

### 3.3 全部混合 + 共享资源 + 容器部署

与上述几种方案最大的不同在于，转让的资源规则是动态决策的。在一个大企业中，服务数量数以万计，要求所有在线服务制定扩缩容决策是很难做到的。同时，业务在部署服务时更关注服务稳定性，常常按照最大流量评估资源，这样就导致流量低峰期有大量的资源浪费。
比较典型的是阿里，百度、腾讯的方案

## 4. 离在线混部的难点和常见方案

### 4.1 资源隔离

为了保证离线业务的运行不能影响在线业务的SLA，那必须要做好离在线资源的隔离,其实K8S本身的QoS策略就是为了按照优先级保障业务的稳定性，当Node节点的资源到达安全阈值的时候会发生[节点压力驱逐](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/)优先驱逐掉BestEffort类型的离线业务，其次是Burstable。但是当资源发生争抢，系统负载高的时候，仅仅通过K8S自身的手段，保证不了在线业务质量，在线业务会有明显的长尾延迟。所以一般会通过一些隔离手段来保证在线业务的稳定性，业界主要有如下一些方式

#### CPU 隔离

- CPU Burst

CPU Burst 技术最初由阿里云操作系统团队提出并贡献到Linux社区和龙蜥社区，分别在 Linux 5.14 和龙蜥ANCK 4.19 版本可用，在传统的 CPU Bandwidth Controller quota 和 period 基础上引入 burst 的概念。当容器的 CPU 使用低于 quota 时，可用于突发的 burst 资源累积下来；当容器的 CPU 使用超过 quota，允许使用累积的 burst 资源。最终达到的效果是将容器更长时间的平均 CPU 消耗限制在 quota 范围内，允许短时间内的 CPU 使用超过其 quota

在容器场景中使用 CPU Burst 之后，测试容器的服务质量显著提升，在实测中可以发现使用该特性技术以后，RT长尾问题几乎消失

[干掉讨厌的 CPU 限流，让容器跑得更快](https://developer.aliyun.com/article/786430?spm=a2c4g.11186623.0.0.48787123wBxE3H)

- cfs 抢占调度

CFS让高优任务占尽优势的同时，也保留了低优任务使用CPU的权利，作为一个通用的调度算法，CFS是公平也是合理的。但当我们但在混部的场景下这会导致一个监控上的毛刺，或引发线上服务的抖动,CFS从设计理念上就不支持绝对抢占，因此，腾讯云原生内核自行研发了支持绝对抢占的离线调度器——BT调度

[云原生下，TencentOS “如意” CPU QoS之绝对抢占](https://cloud.tencent.com/developer/article/1876817)

- L3 Cache隔离技术

英特尔RDT提供了一个由多个组件功能（包括 CMT、CAT、CDP、MBM 和 MBA）组成的框架，用于缓存和内存监控及分配功能。这些技术可以跟踪和控制平台上同时运行的多个应用程序、容器或 VM 使用的共享资源，例如最后一级缓存 (LLC) 和主内存 (DRAM) 带宽。RDT 可以帮助检测“吵闹的邻居”并减少性能干扰，从而确保复杂环境中关键工作负载的性能

[英特尔® 资源调配技术 (英特尔® RDT) 框架](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/resource-director-technology.html)

- 超线程隔离

在某些线上业务场景中，使用超线程情况下的 QPS 比未使用超线程时下降明显，并且相应 RT 也增加了不少。根本原因跟超线程的物理性质有关，超线程技术在一个物理核上模拟两个逻辑核，两个逻辑核具有各自独立的寄存器（eax、ebx、ecx、msr 等等）和 APIC，但会共享使用物理核的执行资源，包括执行引擎、L1/L2 缓存、TLB 和系统总线等等。这就意味着，如果一对 HT 的一个核上跑了在线任务，与此同时它对应的 HT 核上跑了一个离线任务，那么它们之间是会发生竞争的，这就是我们需要解决的问题。

为了尽可能减轻这种竞争的影响，我们想要让一个核上的在线任务执行的时候，它对应的 HT 上不再运行离线任务；或者当一个核上有离线任务运行的时候，在线任务调度到了其对应的 HT 上时，离线任务会被驱赶走。听起来离线混得很惨对不对？但是这就是我们保证 HT 资源不被争抢的机制

[龙蜥Group Identity 技术](https://help.aliyun.com/document_detail/338407.html)

[龙蜥超线程隔离 SMT expeller 技术](https://mp.weixin.qq.com/s/_DTQ4Q2dC-kN3zyozGf9QA)

- 离线大框，动态绑核

通过cpuset cgroup对整体混部大框做了绑核处理，避免混部任务进程频繁切换干扰在线业务进程。当混部算力改变时，再给离线大框动态选取相应的cpu核心进行绑定。同时在选取cpu核心的时候还需要考虑了cpu HT，即尽量将同一个物理核上的逻辑核同时绑定给混部任务使用。否则，如果在线任务和混部任务分别跑在一个物理核的两个逻辑核上，在线任务还是有可能受到“noisy neighbor”干扰

[B站云原生混部技术实践](https://mp.weixin.qq.com/s/pPEkfrLm0XEpgMU1KjiD4A)

#### 内存隔离

- 存带宽分配(MBA)隔离
- 内存提前异步回收
- oom score
- 基于内存安全阈值的主动驱逐机制

#### 磁盘隔离

- 使用分布式存储实现存算分离
- 对块设备执行iops限速
- bufferio 使用cgroupv2限制cache使用量
- 离线业务remote shuffle
- 使用diskquota对文件系统空间和inode数量限制

#### 网络隔离

- tc
- 网络QoS标签
- ipset/iptables

#### 干扰检测

虽然上述讨论了几种主要资源的管理，但由于底层技术限制，部分资源的管理还不完善，并且竞争资源不仅仅是这些，所以，为了保证安全混部，还要增加干扰检测和冲突处理。换句话说，资源隔离是为了避免干扰，干扰检测是根据应用的实际运行情况在冲突发生前或发生后采取措施。在线作业可以提供获取时延数据的方式，或者暴露相关接口，另外还可以收集一些硬件指标，如通过perf收集CPI数据，RDT收集L3 Cache，或epbf收集内核关键路径数据。通过算法分析来判断在线作业是否受影响，若发现异常，便开始降低离线资源，驱逐离线任务

### 4.2 离线业务的资源保障

- 可压缩先压缩
- 不可压缩资源避让足量
- 基于资源满足的驱逐机制

### 4.3 离在线调度框架的整合

k8s的每个节点都会上报节点资源总量（例如allocatable cpu）。对于一个待调度的pod，k8s调度器会查看pod的资源请求配额（例如cpu reqeust）以及相关调度约束，然后经过预选和优选阶段，最终挑选出一个最佳的node用于部署该pod。如果混部任务直接使用这套原生的调度机制会存在几个问题：

1. 混部pod会占用原生的资源配额（例如cpu request），这会导致在线任务发布的时候没有可用资源；

2. 原生调度本质是静态调度，没有考虑机器实际负载，因此没法有效地将混部任务调度到实际负载有空闲的机器上

3. 不支持Gang调度

### 4.4 可观测性体系的建立

可观测性对于排查单机的混部问题非常有用，我们需要监控曲线来查看某个时刻的混部资源使用情况。同时，我们也可以随时查看整体集群的混部算力使用趋势

### 4.5 部门墙

公司内部，机器的产品线一般是固定的，成本和利用率也是按照产品线计算，所以通常情况下机器是不会在不同部门之间自由流转的。引入在离线混部之后，势必需要打破部门墙，对成本和资源利用率计算有一个能融合能分解的调整，才能准确反映出混部的巨大成本价值并持续精细化运营

## 5. Koordinator 介绍

> 在版本迭代过程中，Koordinator社区始终围绕三大能力而构建，即任务调度、差异化 SLO 以及 QoS 感知调度能力。

![picture 1](/images/%E7%A6%BB%E5%9C%A8%E7%BA%BF%E6%B7%B7%E9%83%A8%E8%B0%83%E7%A0%94_koordinator_arch.png)  

### 5.1 任务调度

- Enhanced Coscheduling
  
机器学习和大数据领域中广泛存在的具备 All-or-Nothing 需求的作业负载。例如当提交一个Job 时会产生多个任务，这些任务期望要么全部调度成功，要么全部失败。这种需求称为 All-or-Nothing。为了解决 All-or-Nothing 调度需求，Koordinator 基于社区已有的 Coscheduling 实现了 Enhanced Coscheduling：
支持 Strict/NonStrict 模式，解决大作业场景长时间得不到资源问题
支持 AI 场景多角色联合的 coscheduling 策略，例如一个 TF  训练 Job 中包含 PS 和 Worker 两种角色，并且两种角色都需要单独定义 MinMember，但又期望两种角色的 All-or-Nothing 都满足后才能继续调度

- Enhanced ElasticQuota Scheduling
 
企业的 Kubernetes 一般由多个产品和研发团队共用一个规模比较大的 Kubernetes 集群，由资源运维团队统一管理集群内大量 CPU/Memory/Disk 等资源。
Koordinator 为帮助用户管理好资源额度，提升资源额度的使用效率，实现降本增效，Koordinator 基于基于社区 ElasticQuota CRD 实现了 Enhanced  ElasticQuota Scheduling 

Koordinator ElasticQuota Scheduling 通过额度借用机制和公平性保障机制，Koordinator 把空闲的额度复用给更需要的业务使用。当额度的所有者需要额度时，Koordinator 又可以保障有额度可用。通过树形管理机制管理 Quota，可以方便的与大多数公司的组织结构映射，解决同部门内或者不同部门间的额度管理需求。

- Fine-grained Device Scheduling
 
在 AI 领域，GPU、RDMA、FPGA 等异构资源被广泛的应用，Koordinator 针对异构资源调度场景，提供了精细化的设备调度管理机制，包括：
支持 GPU 共享，GPU 独占，GPU 超卖
支持百分比的 GPU 调度
支持 GPU 多卡调度
NVLink 拓扑感知调度（doing）

### 5.2 差异化 SLO

差异化 SLO 是 Koordinator 提供的核心混部能力，保障资源超卖之后的 Pod 的运行稳定性。Koordinator 定了一一组 Priority & QoS，用户按照这一最佳实践的方式接入应用，配合完善的资源隔离策略，最终保障不同应用类型的服务质量。

- CPU Supress
  
Koordinator 的单机组件 koordlet 会根据节点的负载水位情况，调整 BestEffort 类型 Pod 的 CPU 资源额度。这种机制称为 CPU Suppress。当节点的在线服务类应用的负载较低时，koordlet 会把更多空闲的资源分配给 BestEffort 类型的 Pod 使用；当在线服务类应用的负载上升时，koordlet 又会把分配给 BestEffort 类型的 Pod 使用的 CPU 还给在线服务类应用。

- CPU Burst
 
CPU Burst 是一种服务级别目标 (SLO) 感知的资源调度功能。用户可以使用 CPU Burst 来提高对延迟敏感的应用程序的性能。内核的调度器会因为容器设置的 CPU Limit 压制容器的 CPU，这个过程称为 CPU Throttle。该过程会降低应用程序的性能。
Koordinator 自动检测 CPU Throttle 事件，并自动将 CPU Limit 调整为适当的值。通过 CPU Burst 机制能够极大地提高延迟敏感的应用程序的性能。

- 基于内存安全阈值的主动驱逐机制
 
当延迟敏感的应用程序对外提供服务时，内存使用量可能会由于突发流量而增加。类似地，BestEffort 类型的工作负载可能存在类似的场景，例如，当前计算负载超过预期的资源请求/限制。这些场景会增加节点整体内存使用量，对节点侧的运行时稳定性产生不可预知的影响。例如，它会降低延迟敏感的应用程序的服务质量，甚至变得不可用。尤其是在混部场景下，这个问题更具挑战性

Koordinator 中实现了基于内存安全阈值的主动驱逐机制。koordlet 会定期检查 Node 和 Pods 最近的内存使用情况，检查是否超过了安全阈值。如果超过，它将驱逐一些 BestEffort 类型的 Pod 释放内存。在驱逐前根据 Pod 指定的优先级排序，优先级越低，越优先被驱逐。相同的优先级会根据内存使用率（RSS）进行排序，内存使用率越高越优先被驱逐。

- 基于资源满足的驱逐机制
 
CPU Suppress 在线应用的负载升高时可能会频繁的压制离线任务，这虽然可以很好的保障在线应用的运行时质量，但是对离线任务还是有一些影响的。虽然离线任务是低优先级的，但频繁压制会导致离线任务的性能得不到满足，严重的也会影响到离线的服务质量。而且频繁的压制还存在一些极端的情况，如果离线任务在被压制时持有内核全局锁等特殊资源，那么频繁的压制可能会导致优先级反转之类的问题，反而会影响在线应用。虽然这种情况并不经常发生。

为了解决这个问题，Koordinator 提出了一种基于资源满足度的驱逐机制。我们把实际分配的 CPU 总量与期望分配的 CPU 总量的比值成为 CPU 满足度。当离线任务组的 CPU 满足度低于阈值，而且离线任务组的 CPU 利用率超过 90% 时，koordlet 会驱逐一些低优先级的离线任务，释放出一些资源给更高优先级的离线任务使用。通过这种机制能够改善离线任务的资源需求。

- L3 Cache 和内存带宽分配(MBA) 隔离
 
混部场景下，同一台机器上部署不同类型的工作负载，这些工作负载会在硬件更底层的维度发生频繁的资源竞争。因此如果竞争冲突严重时，是无法保障工作负载的服务质量的。
Koordinator 基于 Resource Director Technology (RDT, 资源导向技术) ，控制由不同优先级的工作负载可以使用的末级缓存（服务器上通常为 L3 缓存）。RDT 还使用内存带宽分配 (MBA) 功能来控制工作负载可以使用的内存带宽。这样可以隔离工作负载使用的 L3 缓存和内存带宽，确保高优先级工作负载的服务质量，并提高整体资源利用率。

### 5.3 QoS 感知调度能力

Koordinator 差异化 SLO 能力在节点侧提供了诸多 QoS 保障能力能够很好的解决运行时的质量问题。同时 Koordinator Scheduler 也在集群维度提供了增强的调度能力，保障在调度阶段为不同优先级和类型的 Pod 分配合适的节点。

- 负载感知调度
 
Koordinator 的调度器提供了一个可配置的调度插件控制集群的利用率。该调度能力主要依赖于 koordlet 上报的节点指标数据，在调度时会过滤掉负载高于某个阈值的节点，防止 Pod 在这种负载较高的节点上无法获得很好的资源保障，另一方面是避免负载已经较高的节点继续恶化。在打分阶段选择利用率更低的节点。该插件会基于时间窗口和预估机制规避因瞬间调度太多的 Pod 到冷节点机器出现一段时间后冷节点过热的情况。

- 精细化 CPU 调度
 
随着资源利用率的提升进入到混部的深水区，需要对资源运行时的性能做更深入的调优，更精细的资源编排可以更好的保障运行时质量，从而通过混部将利用率推向更高的水平。
我们把 Koordinator QoS 在线应用 LS 类型做了更细致的划分，分为 LSE、LSR 和 LS 三种类型。拆分后的 QoS 类型具备更高的隔离性和运行时质量。通过这样的拆分，整个 Koordinator QoS 语义更加精确和完整，并且兼容 Kubernetes 已有的 QoS 语义

[更多参考](https://koordinator.sh/docs)
 
 ### 5.4 总结
    
除了一些隔离特性依赖于内核的支持和缺乏一些运营平台如可观测平台，用户资源画像平台外，Koordinator是一个相对完善的在离线混部的解决方案

## 6. Volcano 介绍

>Volcano是CNCF 下首个也是唯一的基于Kubernetes的容器批量计算平台，主要用于高性能计算场景。它提供了Kubernetes目前缺 少的一套机制，这些机制通常是机器学习大数据应用、科学计算、特效渲染等多种高性能工作负载所需的。作为一个通用批处理平台，Volcano与几乎所有的主流计算框 架无缝对接，如Spark 、TensorFlow 、PyTorch 、 Flink 、Argo 、MindSpore 、 PaddlePaddle 等。它还提供了包括基于各种主流架构的CPU、GPU在内的异构设备混合调度能力

![picture 2](/images/%E7%A6%BB%E5%9C%A8%E7%BA%BF%E6%B7%B7%E9%83%A8%E8%B0%83%E7%A0%94_volcano_arch.png)  

### 6.1 生态支持比较全面
Volcano已经支持几乎所有的主流计算框架：
- Spark
- TensorFlow
- PyTorch
- Flink
- Argo
- MindSpore
- PaddlePaddle
- OpenMPI
- Horovod
- mxnet
- Kubeflow
- KubeGene
- Cromwell

### 6.2 丰富的调度策略
Volcano支持各种调度策略，包括：

- Gang-scheduling
- Fair-share scheduling
- Queue scheduling
- Preemption scheduling
- Topology-based scheduling
- Reclaims
- Backfill
- Resource Reservation

得益于可扩展性的架构设计，Volcano支持用户自定义plugin和action以支持更多调度算法。

### 6.3  增强型的Job管理能力
Volcano提供了增强型的Job管理能力以适配高性能计算场景。这些特性罗列如下：

多pod类型job
增强型的异常处理
可索引Job

### 6.4 异构设备的支持
Volcano提供了基于多种架构的计算资源的混合调度能力：
- x86
- ARM
- 鲲鹏
- 昇腾
- GPU

### 6.5 总结
Volcano支持丰富的离线计算生态，对现有大数据和AI计算调度场景天生兼容比较好，但其发展重点侧重于解决调度问题，作为离在线整体的解决方案还需要自己补充很多其他特性 

[更多参考](https://volcano.sh/zh/docs/)

## 7. 总结：实践思路

线上：分时复用 + 独占资源 + 动态调度

![picture 3](/images/%E7%A6%BB%E5%9C%A8%E7%BA%BF%E6%B7%B7%E9%83%A8%E8%B0%83%E7%A0%94_colocation_online_arch.png)  

线下：共享内核 + 容器部署 + 动态策略

![picture 4](/images/

![picture 5](/images/colocation_investigate_colocation_offline_arch_new.png)  

## 参考文档

[1] [腾讯：Caelus—全场景在离线混部解决方案](https://cloud.tencent.com/developer/article/1759977)

[2] [字节跳动：自动化弹性伸缩如何支持百万级核心错峰混部](https://cloud.tencent.com/developer/news/653586)

[3] [字节跳动：混布环境下集群的性能评估与优化](https://www.intel.cn/content/www/cn/zh/customer-spotlight/cases/bytedance-performance-evaluation-optimization.html)

[4] [百度大规模战略性混部系统演进](https://www.infoq.cn/article/aeut*zaiffp0q4mskdsg)

[5] [百度混部实践：如何提高 Kubernetes 集群资源利用率？](https://mp.weixin.qq.com/s/12XFN2lPB3grS5FteaF__A)

[6] [阿里巴巴：数据中心日均 CPU 利用率 45% 的运行之道--阿里巴巴规模化混部技术演进](https://developer.aliyun.com/article/651202)

[7] [阿里巴巴：Co-Location of Workloads with High Resource Efficiency](https://static.sched.com/hosted_files/kccncosschn19chi/70/ColocationOnK8s.pdf)

[8] [阿里大规模业务混部下的全链路资源隔离技术演进](https://mp.weixin.qq.com/s/_DTQ4Q2dC-kN3zyozGf9QA)

[9] [一文看懂业界在离线混部技术](https://www.infoq.cn/article/knqswz6qrggwmv6axwqu)

[10] [美团：提升资源利用率与保障服务质量，鱼与熊掌不可兼得？](https://mp.weixin.qq.com/s/hQKM9beWcx7CKMvpJxznfQ)

[11] [B站云原生混部技术实践](https://mp.weixin.qq.com/s/pPEkfrLm0XEpgMU1KjiD4A)

[12] [华为：基于Volcano的离在线业务混部技术探索](https://www.bilibili.com/video/BV1AZ4y1X7AQ/?vd_source=cad2cc6310088fc3945e9d1cb002adee)

[13] [K8S节点压力驱逐](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/)

[14] [摆脱不必要的节流,同时实现高CPU利用率和高应用程序性能](https://www.youtube.com/watch?v=9ao7Ix2ugK4)
