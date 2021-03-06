---
title: 跨云应用管理方案分析
date: 2018-11-14 00:14:39
categories:
- 架构
tags:
- Kubernetes
---

跨云应用管理方案分析
## 需求背景
随着技术的不断发展，越来越多的产品通过使用分布式架构，来解决服务器压力、数据可靠性的问题。为了更聚焦于自己的核心业务，许多企业会直接采用各类云厂商的服务或方案，降低门槛与成本。
但是对于那些业务灵活、数据敏感的企业及产品，使用的云计算服务就需要具有跨云厂商、灵活伸缩、快速切换等功能，保证在服务变更、功能调整等场景下业务受到较小冲击。

## 相关技术
### 云计算
云计算是一种信息技术（IT）模式，可以随时访问共享的可配置系统资源池和更高级的服务，这些服务通常可以通过互联网以最少的管理工作快速供应。云计算依靠资源共享来实现连贯性和规模经济，类似于公用事业。

### 容器化
容器占用资源少、部署快，每个应用可以被打包成一个容器镜像，每个应用与容器间成一对一关系也使容器有更大优势，使用容器可以在build或release的阶段，为应用创建容器镜像，因为每个应用不需要与其余的应用堆栈组合，也不依赖于生产环境基础结构，这使得从研发到测试、生产能提供一致环境。类似地，容器比虚机轻量、更“透明”，这更便于监控和管理。

### Kubernetes
又称K8S，是Google开源的一个容器编排引擎，它支持自动化部署、大规模可伸缩、应用容器化管理。在生产环境中部署一个应用程序时，通常要部署该应用的多个实例以便对应用请求进行负载均衡。
在K8S中，我们可以创建多个容器，每个容器里面运行一个应用实例，然后通过内置的负载均衡策略，实现对这一组应用实例的管理、发现、访问，而这些细节都不需要运维人员去进行复杂的手工配置和处理。

## K8S集群
K8S通过将多个主机组成集群并在上层再进行一次虚拟化，使得资源可以充分被利用并灵活地调度。
但是K8S集群是基于局域网建立的，所以若使用云主机的话，单个集群通常无法跨单个云厂商的多个区域，更不用说支持不同的云厂商。这个对于容灾、迁移、管理方面都是很大的制约。

## K8S联邦集群（Federation）
K8S提供了联邦集群的机制，将分布在多个区域或者多个云厂商的K8S集群整合成一个大的集群，统一管理与调度。

### 优点
- 低延迟：让多个区域中的集群通过向距离它们最近的用户提供服务来最大限度地减少延迟。
- 故障隔离：最好有多个小型集群而不是一个单独的大型集群来进行故障隔离。
- 可扩展性：单个K8S集群具有可扩展性限制。
- 混合云：可以在不同的云提供商或本地数据中心上拥有多个群集。

### 实现
联邦集群可以轻松管理多个群集，它通过2个主要构件来实现：
- 跨群集同步资源：联邦可以使多个群集中的资源保持同步。 例如，可以确保多个群集中部署相同的程序。
- 跨群集发现：联邦提供了自动配置DNS服务器和对群集后端负载均衡的功能
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/k8s_federation_architecture.jpg)

### 影响
虽然联邦有很多有吸引力的用处，但也有一些注意事项：
- 增加网络带宽和成本：联邦控制台监视所有群集以确保当前状态符合预期。如果集群在云提供商或不同云提供商的不同区域(regions)运行，这可能会导致显著的网络成本。
- 减少跨群集隔离：联邦控制台中的错误可能影响所有群集。通过将联邦控制台中的逻辑保持最简，可以缓解这一问题，尽可能将大部分任务都交给K8S集群处理。 设计和实施也在安全方面做了很多考虑，并避免发生错误时多集群停机。
- 成熟度：联邦项目相对较新，不太成熟。 并非所有资源都可用，许多资源仍然是alpha状态。


## 跨云集群方案
### 框架
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/%E8%B7%A8%E4%BA%91%E5%B9%B3%E5%8F%B0%E6%9E%B6%E6%9E%84%E6%A1%86%E6%9E%B6%E5%9B%BE.jpg)
简单框架原型，镜像功能独立于跨云，暂不列入框架图

#### 业务层
K8S联邦集群可以管理的资源如框架图中所示，基于这些资源我们可以提供如下跨云功能：
- 集群管理
- 部署
- 服务
- 负载均衡
- 资源隔离
- 冗余/可靠性
- 安全

#### 平台层
利用K8S联邦集群的控制平台，运行一个核心管理平台，衔接业务层的各类操作和底层每个云集群的资源控制动作。

#### 服务器层
通过在各个云集群部署代理应用，获取各个云集群及下属云主机的控制权、资源、日志等等


### 云主机
云主机可以由客户采购+平台管理或者平台采购+平台管理：
- 客户采购+平台管理：由客户自行采购主机并管理核心业务，我们提供平台或工具进行管理，节省了集群采购开支并降低产品运营、数据相关的政策风险。但是至少需要获得用户的主机管理权限才能进行管理，对于一些业务敏感的企业可能会有抵触。
- 平台采购+平台管理：由我们采购大量主机并搭建平台，类似于给用户提供PaaS服务，增加了主机采购维护的成本和风险，好处在于具有完整的主机控制管理权，可以更自如地部署平台层面的资源调度管理功能。

### 规格
基于K8S v1.12版本，每个集群可以管理的规格如下：
- 不超过5000个nodes
- 不超过150000个pods
- 不超过300000个容器
- 每个node上不超过100个pods


## 市场、竞品分析
应用交付(Application Delivery Controller，简称ADC)主要部署于数据中心的应用服务器前端，利用负载均衡、服务器卸载、压缩和缓存、链接复用和防火墙等技术，实现应用的可用性、高效性和安全性。
主要厂商：深信服、H3C、F5、A10、Radware和Array等

### F5
行业领导者，F5 解决方案集成了以下各项：
- 公有云提供商，如 AWS、Microsoft Azure 和 Google Cloud Platform。
- 私有云和开源平台，如 OpenStack、VMware 和 OpenShift。
- 自动化工具包，因此您可以使用 Ansible、Puppet 或 HashiCorp 部署应用服务，在 DevOps 方法中为应用服务提供声明接口。
- 容器环境中的微服务（如 Kubernetes 和 Mesos），以便在东西向流量中启用应用服务。

### A10 Network 
#### Harmony Controller 
Harmony 平台的理念是随时随地实施策略，它将安全应用服务与内部部署环境、公有云、私有云和混合云衔接在一起。A10 Harmony 控制器具备虚拟化、SaaS 和物理规格，可用于管理多云环境下的安全应用交付服务，并集成多种配置和自动化服务。


> 参考资料：  
> [Federation - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/federation/)  
> [Building Large Clusters - Kubernetes](https://kubernetes.io/docs/setup/cluster-large/)  
> [A10 Networks来了，给你来点不一样的料 - 51CTO.COM](http://stor.51cto.com/art/201708/546850.htm)  
> [专访：多云应用交付新时代下的F5与中国-云计算专区](http://cloud.it168.com/a2018/0619/3210/000003210034.shtml)  
> [F5发布2018全球应用交付报告](http://network.51cto.com/act/f5/ADreport)  

