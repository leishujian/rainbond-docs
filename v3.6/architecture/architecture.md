---
title: Rainbond技术架构
summary: Rainbond技术架构及组件组成部分
toc: true
---

## 一、Rainbond总体技术架构

<img width="100%" src="https://static.goodrain.com/images/docs/3.6/architecture/architecture.png"></img>

Rainbond践行以应用为中心的理念，吸纳优秀的社区解决方案，形成了应用控制、应用运行时，集群控制三大模块结合的数据中心技术架构，结合跨数据中心的上层结构应用控制台和资源控制台，形成了完整的PaaS平台解决方案，下面将对每个组件和重点流程进行介绍：

## 二、Chaos组件之应用构建（CI）

<img src="https://static.goodrain.com/images/docs/3.6/architecture/app-ci.png" width="100%" />

​Rainbond 应用构建(CI)的过程是将输入介质：源代码 或 Docker镜像进行一系列处理，最终生成 `应用抽象` 介质的过程。

此过程可重复操作，从而生成一系列的应用版本。

传统意义上说，完整的CI过程会包括：设计、编码、打包、测试和发布，Docker自推出以来逐步成为众多应用代码打包的新形式。

现有的CI产品中已经在源码测试和Pipline方面做得非常成熟，例如Jenkins，Gitlab等，因此Rainbond在对于源码或Docker镜像的前置处理方面可以直接对接第三方服务，由第三方服务处理完成的源码或镜像再对接到 Rainbond-Chaos 模块进行应用抽象。

这样Chaos的输入源是支持Git协议的代码仓库，Docker镜像仓库。如果是源代码，Chaos智能判断源码的类型，例如Java,PHP,Python，Dockerfile等，根据不同的源码类型选择对应的BuildingPack进行源码编译，同时识别源码中定义的运行环境要求参数，应用端口、环境变量等参数，形成应用抽象的配置雏形。除了Dockerfile以外的源码类型将被编译成应用代码环境包（SLUG）存储于分布式存储中，其他的生成Docker本地镜像存储于数据中心镜像仓库中，结合应用的各类属性信息形成`应用抽象包`。

> - 关于源码编译的BuildingPack参考[各语言支持文档](../user-manual/language-support/java.html)。
> - Chaos组件支持多点高可用部署，多点部署从MQ组件获取应用构建任务。

## 三、Worker组件之应用部署（CD）

<img src="https://static.goodrain.com/images/docs/3.6/architecture/app-cd.png" width="100%" />

worker组件将chaos构建出来的应用抽象进行实例化，配属应用运行需要的各类资源，完成应用生命周期中的运行态部分（CD）。

应用生命周期中可能经历启停、升级与回滚。不同的应用类型需要进行不同的控制策略，例如无状态应用能够进行无序的滚动升级，而有状态应用的升级控制策略将更加复杂。

应用运行需要各种外部环境支持，例如需要分配租户IP,端口等网络资源，需要根据应用设置配属持久化存储资源，例如共享文件存储分配存储目录，块存储等依托各类插件分配存储资源。根据应用依赖属性建立服务发现和负载均衡策略供给mesh插件。根据应用属性生成调度策略调用Kubernetes集群调度应用运行。

目前Rainbond使用Kubernetes以下资源类型：ReplicationController、Deployment、Statefulset、Service、Pod。对于用户来说，无需理解这些资源，我们产品中也不体现，其只是应用运行的载体。

> worker组件功能分为有状态部分和无状态部分，为了实现worker组件的集群部署，worker进行了主节点选举，当选主节点的服务将额外启动一个gRPC服务，提供应用状态等数据服务

## 四、Entrance组件之应用负载均衡（LB）

<img src="https://static.goodrain.com/images/docs/3.6/architecture/rbd-entrance.png" width="100%" />

当应用成功运行后，面临的就是如何访问的问题，下面就详细讲解Rainbond在应用运行后所做的处理：

- 应用运行时会为每个租户分配子网
  - 租户之间网络隔离，因此集群内运行的应用不能之间被外网访问到。
  - 应用每次启动IP地址跟随变化，所以租户内应用与应用直接也无法直接访问。
- 配置应用端口策略，平台设计了应用端口的服务控制，具有两种类型
  - 对内服务：对平台内部应用开放，并加入到租户级别内部负载均衡，内部应用间访问通过ServiceMesh实现
  - 对外服务：对平台外部应用开放，并加入到平台全局负载均衡
- 设置应用协议类型，应用支持HTTP和TCP协议的应用，并支持添加到负载均衡

> Entrance组件支持集群部署，并通过etcd实现全局资源一致性，可防止了对同一个资源操作的重复进行。

## 五、Eventlog组件之应用日志处理
​
Rainbond平台需要处理的日志和消息信息包含三大类，分别为：用户异步操作日志、应用构建日志和应用运行日志，下面针对这三类日志加以详细说明。

### 5.1 用户异步操作日志

对于用户操作日志，我们需要分布式跟踪每一次操作的最终状态，目前由eventlog组件根据每一次操作的日志汇聚判断操作的最终状态。其他组件在处理某一次异步任务过程中会将过程日志记录通过gRPC消息流发送到eventlog集群。

### 5.2 应用构建日志

主要展示源码构建或Docker镜像构建的输出信息，这些信息来自于Chaos组件。

### 5.3 应用运行日志等

关于应用日志，目前我们分为两种形式：

- (1) 标准输出/错误输出的日志
> 对于标准输出的日志，Rainbond定制了docker日志处理驱动插件，基于TCP数据流通信实现将所有计算节点的容器日志实时送往Eventlog组件按照应用级别的汇聚，从而进行存储和实时推送到UI。

- (2) 输出到持久化文件的业务日志（访问日志）
> 对于输出到持久化目录的业务日志，一般需要对其进行自动分析，例如对接ELK系统，因此在插件体系中安装日志处理插件，收集持久化目录的日志文件输送到第三方日志分析服务上。


随着集群规模越大，运行应用越多，日志处理量非常大，因此我们实现了Eventlog的集群，每一个应用的日志在传输之前会选择送往的eventlog服务节点，类似于数据分区。选择过程中做了均衡分配处理，例如当前有10000个应用，3个eventlog服务节点，将做到每个eventlog节点分别处理3000左右应用日志。

> 由于各种实时推送的需要，eventlog组件实现了websockt服务。

## 六、Webcli组件之应用容器控制

该组件实现了通过web的方式连接到容器控制台的功能。该组件通过与UI进行WebSocket通信，用户可以通过模拟Web终端发送各类shell命令，webcli通过kubernets提供的exec方式在容器中执行命令并返回结果到Web终端。

> Webcli属于无状态组件，天然支持多点高可用部署。

## 七、Monitor组件之集群监控

Rainbond集群需要多个维度的监控：

- 应用业务性能级
- 应用容器资源级
- 集群节点级
- 管理服务级

Rainbond借助[Prometheus](https://prometheus.io/)封装了Monitor组件，能够自动发现以上描述的各类监控对象并完成配置，将监控目标纳入Prometheus监控范围。Rainbond各组件也都实现了Prometheus的exporter端暴露监控指标。

Prometheus有单点性能障碍，例如单节点服务监控目标越多，内存与磁盘使用量大等问题，对于大型集群，我们借助monitor组件实现了Prometheus多点数据分区运行，并建立了Prometheus的集群查询机制。

除了监控我们还需要报警，monitor能够自动配置一些默认的报警规则，自定义的报警规则支持将在资源管理后台体现，后续将支持命令行控制。Prometheus发出报警信息到monitor,完成去重，忽略等操作后根据级别与用户需求完成邮件、微信、站内消息等报警方式。

## 八、Node组件之集群节点管理与ServiceMesh

<center><img width="85%" src="https://static.goodrain.com/images/docs/3.6/architecture/rainbond-cluster-ctl.png" /></center>

Node组件是Rainbond集群组建的基本服务，集群内所有节点都需要运行该组件。

运行于管理节点的Node具有master角色，维护所有节点的状态与健康检查。在监控方面，Node暴露出节点的操作系统级别各类指标（集成promethes node-exporter），同时定时检查不同属性的节点上运行的各类服务状态，网络状态等。监控到问题并尝试自动处理，以提供集群自动化运维能力。

​第二方面，所有计算节点运行的Node服务组建起租户网络内运行应用的运行环境支持，特别是ServiceMesh支持。

<center>
<img width="95%" src="https://static.goodrain.com/images/docs/3.6/architecture/ServiceMesh.png" /></center>

Node提供了envoy的全局化配置发现支持，跟应用绑定的envoy插件通过宿主机网络跳出租户网络，访问到Node服务，并最终获取全局的服务网络治理配置。其他应用插件使用同样的机制从node服务中动态获取配置，应用运行信息等。

## 九、MQ消息组件

​MQ组件是基于etcd实现的轻量级分布式、消息持久化和一致性的消息中间件。该组件维护异步任务消息，提供多种主题的发布和订阅能力。

我们没有选择使用已有的消息中间件服务，主要是我们要实现分布式消息，还要保证消息的一致性，评估了现有的方案后都觉得不太合适且复杂，因此我们基于etcd的分布式能力实现了轻量级MQ组件，提供了gRPC和http两种接口实现pub/sub。

## 十、App-UI组件

应用控制台UI组件作为Rainbond以应用为中心抽象的关键模块，面向应用开发者和应用使用者。基于Django+Ant design前后端分离架构设计，提供用户对应用抽象、应用组抽象，数据中心抽象，应用市场抽象提供交互体验。实现完整的应用创建、管理流程，应用交付分享流程等。

## 十一、Resource-UI组件（企业版）

资源控制台UI组件提供Rainbond集群资源管理，计划支持对接IaaS的资源管理能力，面向运维人员设计。关注节点物理资源，集群资源，管理服务资源，应用实际使用资源，租户资源等管理。Rainbond自动化运维能力的关键展示平台。

## 十二、命令行工具grctl

grctl命令行工具提供一些有趣实用的应用管理功能和集群运维功能，并逐步丰富中。对于开源使用用户来说在没有ResourceUI的情况下方便进行集群管理和运维。