---
layout: post
title: "容器化迁移规范"
date: 2018-11-28
categories: 容器
---

## 服务器选择
用户自行采购服务器主机，服务器规格要求如下：
- 操作系统：CentOS 7或RHEL 7
- 规格：为了流畅使用Kubernetes功能，主机规格至少达到2 Cpu + 2GB内存
- 区域：不限，根据用户业务自定
- 带宽：不限，根据用户业务自定
- 安全组：需要放开如下Kubernetes集群使用到的端口
	- Kubernetes API server：6443*
	- etcd server client API：2379-2380
	- Kubelet API：10250
	- kube-scheduler：10251
	- kube-controller-manager：10252
	- NodePort Services：30000-32767
- 其他：保证主机的MAC地址和product_uuid在集群内是唯一的；服务器主机之间的网络可以完全正常访问


## 集群搭建
集群由至少一台Master加若干Node节点组成，如果需要实现更高可靠性，可以增加更多服务器。Kubernetes提供了较高的灵活性，后续根据业务扩张情况弹性增加服务器也十分方便。
选定好Master之后，即可使用我们提供的工具搭建集群了。


## 应用镜像制作
容器化的关键步骤之一就是将应用的部署过程制作成镜像，通过分发镜像完成应用的快速实例化、自动化部署的目的。
制作镜像需要关注如下几个关键要素：

### 系统环境
选择应用需要使用的基础镜像，基础镜像通常是各种Linux发行版的Docker镜像比如ubuntu、Debian、centos等，也有已安装好相应软件的基础镜像，如NGINX、Python、redis等。根据用户的需求自行选择对应的系统及版本。
常用的基础镜像可以在官方仓库（https://hub.docker.com/explore）或其他源（[开发者平台](https://dev.aliyun.com)）查询获取。

### 应用依赖 
通过依赖清单的方式明确的声明所有的依赖项，并且在运行过程中，使用依赖隔离工具来确保程序不会调用系统中存在但清单中未声明的依赖项。这样可以简化环境配置流程，并保证应用的生产和开发环境更加一致。
具体到语言上来说例如Ruby 的 Bundler 使用 Gemfile 作为依赖项声明清单，使用 bundle exec 来进行依赖隔离。Python 中则可分别使用两种工具： Pip 用作依赖声明， Virtualenv 用作依赖隔离。甚至 C 语言也有类似工具， Autoconf 用作依赖声明，静态链接库用作依赖隔离。无论用什么工具，依赖声明和依赖隔离建议一起使用。

### 分离有状态与无状态服务
有状态的服务牵扯到存储卷挂载等问题很难在水平方向（服务器主机层面）上进行迁移，所以任何需要持久化的数据都应存储在后端服务（比如数据库），而非直接和服务端耦合在一起，而且数据库等有状态服务也不适合做进镜像内，制作进镜像的部分应该都为无状态且无共享的。
具体来说，通过使用云数据库或者其他方式，把有状态的模块进行服务化，服务端代码只需要引用地址或访问相应API即可使用相关资源。如此实现之后，服务端镜像化之后可以自由地在各个主机上进行实例化，不受数据持续化等问题的制约。

### 记录部署过程
进行一次应用部署，并将部署过程中的如下指令操作记录下来：
- 软件安装
- 配置操作
- 选择暴露的端口

以下面这个Dockerfile为例：
- 通过ENV指令设置环境变量
- 通过ADD指令将本地文件拷贝到容器里
- 通过RUN指令执行命令行操作，这里主要是yum install进行依赖软件包安装
- 通过EXPOSE指令指定需要暴露的服务端口
- 通过ENTRYPOINT指令设置容器运行起来后需要执行的程序或命令
```
FROM daocloud.io/centos:7 
MAINTAINER mate
 
ENV TZ "Asia/Shanghai" 
ENV TERM xterm 

ADD aliyun-mirror.repo /etc/yum.repos.d/CentOS-Base.repo 
ADD aliyun-epel.repo /etc/yum.repos.d/epel.repo 
RUN yum install -y curl wget tar bzip2 unzip vim-enhanced passwd sudo yum-utils hostname net-tools rsync man && \ 
	yum install -y gcc gcc-c++ git make automake cmake patch logrotate python-devel libpng-devel libjpeg-devel && \ 
	yum install -y --enablerepo=epel pwgen python-pip && \ 
	yum clean all 
RUN pip install supervisor
 
ADD supervisord.conf /etc/supervisord.conf 
RUN mkdir -p /etc/supervisor.conf.d && mkdir -p /var/log/supervisor 
EXPOSE 22 
ENTRYPOINT ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"] 
```

备注：
制作镜像过程中，配置可以通过ENV直接设置到环境变量中，但如果是运行的程序依赖的配置需要文件化，类似样例中的supervisord.conf配置文件，那么此文件最好

### 编写Dockerfile
更加详细具体的Dockerfile的语法格式、编写规范可以参考如下资料：
[| Docker Documentation](https://docs.docker.com/engine/reference/builder/)
http://www.docker.org.cn/dockerppt/114.html
