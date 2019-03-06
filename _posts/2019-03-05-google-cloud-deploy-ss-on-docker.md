---
title: 在Google云平台上通过Docker快速搭建ShadowSocks服务
date: 2019-03-05 22:46:49
categories:
- 云计算
- 容器
tags: 
- ShadowSocks
- Docker
- Google
---

## 准备
- 已激活的Google云账号
- 支付方式已经配置成功

## 创建实例
通过Google Cloud Platform左上角的导航菜单进入“Compute Engine - VM实例视图”创建VM实例
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/1.png)

### 选择区域与配置
目前谷歌云支持18个区域,列表如下:
- Taiwan(asia-east1)
- Hong Kong(asia-east2)
- Singapore(asia-southeast1)
- Tokyo(asia-northeast1)
- Montreal, Canada(northamerica-northeast1)
- Los Angeles, USA(us-west2)
- South Carolina, USA(us-east1)
- Northern Virginia, USA(us-east4)
- Sydney(australia-southeast1)
- Mumbai(asia-south1)
- Belgium(europe-west1)
- Netherlands(europe-west4)
- Hamina, Finland(europe-north1)
- London, UK(europe-west2)
- Iowa, USA(us-central1)
- Oregon, USA(us-west1)
- São Paulo(southamerica-east1)
- Frankfurt, Germany(europe-west3)

根据业务选择适合自己的区域,通常选用访问延迟较低的区域,可以根据谷歌云主机测速网站[GCP ping](http://www.gcping.com/)的数据进行选择.
此处以台湾服务器为例
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/2.png)

### (可选)是否使用容器功能
Google云已经全流程支持容器功能,可以在创建实例的时候就进行容器配置
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/3.png)


### 选择映像
如果使用容器化部署的话,系统会自动启用容器定制系统,用户就无需自己再选择映像了.
若没有使用到云平台配套的容器功能,用户可以根据需求选择合适自己的映像(即镜像)
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/4.png)

### 防火墙配置
勾选允许HTTP和HTTPS流量的选项
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/5.png)

## 连接实例
Google Cloud提供了浏览器登录、SSH登录等多种方式,选择合适自己的即可.

## 手工使用Docker部署ShadowSocks服务
使用Google Cloud自带的容器功能可以省去一些安装配置动作.
但是手工部署可以带来更高的灵活性,用户可以选择更多的可选配置.
安装完docker相关组件之后,使用如下命令即可部署SS服务:
`docker run -dt --name ss -p <host_port>:<docker_port> mritd/shadowsocks -s "-s 0.0.0.0 -p <s> -m chacha20-ietf-poly1305 -k <password> --fast-open"`

参数说明:
- -d: 后台运行容器，并返回容器ID；
- -t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
- -p: 根据“服务器端口:容器端口”格式顺序配置端口映射关系
- -s: 用户自行指定参数
	- -s  服务主机IP
	- -p 对外暴露的服务端口
	- -m 选择加密方式,这里选择chacha20-ietf-poly1305
	- -k 配置密码

命令执行完成之后可以通过`docker ps`命令查看容器是否正常运行,
使用`ps -ef | grep ss`命令可以查看ShadowSocks服务是否正常运行.

至此,用户就可以根据自己使用的终端安装对应的客户端使用ShadowSocks服务了.

## 常见问题
- ShadowSocks安装及配置没有错误的情况下，客户端为何还是无法使用服务？
使用云服务器出现这种情况很可能是因为云厂商的网络安全策略限制所致，请确保相应的网络端口都已经打开。此处以谷歌云为例，导航菜单-VPC网络-防火墙规则-创建防火墙规则-填写允许访问的协议和端口。
![](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/20190305/6.png)

> tips 如何快速判断是因为网络端口没有放开导致的服务不可用？  
> 有时用户无法确定到底是软件安装配置还是网络端口问题导致的服务不可用。只需如下一条命令即可快速启用一个web服务`docker run -p <hostPort>:80 -d nginx`，通过访问hostIP:hostPort查看网页是否能够正常访问即可判断端口是否正常放开。  

