---
title: 跨云平台实施方案预研
date: 2018-11-20 15:45:20
category: 架构
tags:
- 云计算
---

## 基于K8S集群搭建
在不同云厂商或同个云厂商不同区域下创建K8S集群，每个K8S集群中独立运行代理程序，通过代理程序进行镜像部署、服务发现、负载均衡等功能。
然后再开发一套对K8S集群进行控制管理的系统作为后台，用来进行多个集群的资源管理调度。
优势：单个K8S集群下的功能可以利用之前拾云平台的现成代码
缺陷：对多个K8S集群的资源进行重新分配管理需要较巨大的开发工作量

此方案工作量较大且重复。

## 基于K8S联邦集群搭建
使用K8S联邦集群的机制，利用K8S定义好的各类资源概念，省去了资源调度、集群间通信、等诸多开发工作量。
![](https://github.com/ShoreCN/ShoreCN.github.io/blob/master/resource/%E8%B7%A8%E4%BA%91%E5%B9%B3%E5%8F%B0%E6%9E%B6%E6%9E%84%E6%A1%86%E6%9E%B6%E5%9B%BE.jpg?raw=true)

可以将产品分为如下几个模块：
- 服务器适配：
这层主要内容是通过在云主机上常驻一个agent程序，对接我们管理平台的控制指令，并实时采集系统资源情况。
可以使用golang开发，具有语法简单性能较高，并且编译生成出服务代理agent程序之后，可以方便地部署在不同的云厂商主机上，不用关心跨厂商主机之间的环境差异等问题。

- 管理平台：
通过将K8S的各类资源（Cluster、Deployment、ReplicasSet、Service、Ingress等），按照业务逻辑封装成我们自己的系统后台功能（集群管理、部署、冗余、服务、负载均衡等），通过沿用部分原有的后台代码，使用Python语言以Tornado为框架开发一套针对上述资源的RESTful API，可以沿用之前的后台框架设计。
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/TenCloud_Architecture.png)
数据层、镜像、用户、Tornado框架部分可以沿用原有代码，重新开发业务服务部分的集群、部署、服务功能。

- 管理平台操作界面：
通过对接后台提供的RESTful API，产品化设计后以前端形式展现管理界面。

### 精简版本
一个可以考虑的精简版本就是开发一个直接跑在云主机上的脚本式命令行管理系统。封装K8S联邦集群的部署过程、模板化整合K8S联邦集群底层的资源操作。
精简版本的开发工作量主要集中在：
- 业务模型梳理封装
- 命令解析转换

问题：
- 涉及一些用户第三方授权的操作（如GitHub OAuth认证），在命令行端可能难以实现
- 若想收集K8S以外的主机、任务等信息，还是需要实现一套响应网络请求的框架

## 基于现有产品搭建
### F5
行业领导者，F5 解决方案集成了以下各项：
- 公有云提供商，如 AWS、Microsoft Azure 和 Google Cloud Platform。
- 私有云和开源平台，如 OpenStack、VMware 和 OpenShift。
- 自动化工具包，因此您可以使用 Ansible、Puppet 或 HashiCorp 部署应用服务，在 DevOps 方法中为应用服务提供声明接口。
- 容器环境中的微服务（如 Kubernetes 和 Mesos），以便在东西向流量中启用应用服务。

### A10 Network 
跨云产品Harmony Controller 
Harmony平台的理念是随时随地实施策略，它将安全应用服务与内部部署环境、公有云、私有云和混合云衔接在一起。A10 Harmony 控制器具备虚拟化、SaaS 和物理规格，可用于管理多云环境下的安全应用交付服务，并集成多种配置和自动化服务。

### RedHat
OpenShift容器平台
红帽 OpenShift 是一款开源容器应用程序平台，主要以 Docker 容器为基础，并采用 Kubernetes 容器集群管理进行编排。OpenShift可支持多种编程语言和服务，包括 Web 框架、数据库或与移动应用和外部后台的连接器。OpenShift 平台同时支持云原生、无状态的应用程序以及传统、有状态的应用
程序。
![](https://github.com/ShoreCN/ShoreCN.github.io/blob/master/resource/Redhat_Openshift_Architecture.jpg?raw=true)

### Rancher
Rancher为DevOps工程师提供了一个直观的用户界面来管理他们的服务容器，用户不需要深入了解Kubernetes概念就可以开始使用Rancher。 Rancher包含应用商店，支持一键式部署Helm和Compose模板。Rancher通过各种云、本地生态系统产品认证，其中包括安全工具，监控系统，容器仓库以及存储和网络驱动程序。
![](https://github.com/ShoreCN/ShoreCN.github.io/blob/master/resource/Rancher_Architecture.jpg?raw=true)

**容器平台能同时对多个Kubernetes集群进行管理，但每个集群上运行的应用和其他集群隔离。**

### 时速云
企业级容器 PaaS 平台 TenxCloud Enterprise (TCE)
随着业务的发展，客户通常会使用多个不同的节点区域，甚至使用多个不同的基础云服务提供商。时速云的跨云管理解决方案通过创建私有集群，并添加不同IaaS的云主机，可一键将容器镜像发布到不同的IaaS平台上，屏蔽了不同IaaS厂商之间的底层异构问题，极大的降低了IT管理的复杂度，并可轻松实现应用在多个不同的IaaS之间的迁移、容灾以及热备份。

方案特点：
- 一键创建私有Docker主机集群，并支持将云主机、物理机或者虚拟机等添加到私有集群。
- 通过时速云容器云平台统一管理多个不同IaaS上的应用。
- 利用容器技术屏蔽底层基础云服务之间的差异，实现应用在多个云之间的迁移、容灾及热备。
- 私有集群可以无缝使用容器部署、镜像下载、集群 API 等服务。

### Cloudify
Cloudify是一个开源的云应用编排系统，可以让你的应用自动化在各种不同的云上方便地部署。Cloudify重点关注应用自动化，承担了部分业务自动化的工作。从云IaaS、PaaS、SaaS分层看，Cloudify是一个典型的面向应用编排自动化的PaaS平台。


> 参考资料  
> [红帽 OPENSHIFT 容器平台 3.5](https://www.redhat.com/cms/managed-files/cl-openshift-container-platform-3.5-datasheet-f7213kc-201704-a4-zh.pdf)  
> [跨集群服务——如何利用Kubernetes 1.3实现跨区高可用 - 简书](https://www.jianshu.com/p/0622d7dbbaa7)  
> [1 - 架构设计 | Rancher Labs](https://www.cnrancher.com/docs/rancher/v2.x/cn/overview/architecture/)  
> [Cloudify：打通应用和基础架构自动化交付的“任督二脉” - liukuan73的专栏 - CSDN博客](https://blog.csdn.net/liukuan73/article/details/57082617?utm_source=blogxgwz3)  
> [Cloud & NFV Orchestration Based on TOSCA | Cloudify](https://cloudify.co/)  


