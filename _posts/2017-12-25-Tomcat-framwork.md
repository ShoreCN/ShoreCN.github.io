---
title: Tomcat架构及运行原理
date: 2017-12-25 19:28:11
categories:
- 架构
tags: 
- 后端
---

## Tomcat架构

Tomcat是一个开源的轻量级Web应用服务器，是Apache软件基金会（Apache Software Foundation）的一个核心项目。

Tomcat的架构核心部分是：连接器（Connector）和容器（Container）。
但从全局视角来看，从下往上Tomcat的架构可以分为以下几个部分：

1. Server: Tomcat运行的服务器实例;
2. Service: 对外提供服务的主体, 一个Server可以包含多个Service;
3. Connector: 连接器，用于处理客户端请求, 一个Service可以包含多个Connector;
4. Container: 容器，用于管理Servlet的生命周期, 一个Service包含一个Container。

![tomcat_framwork](/resource/tomcat_framwork.png)

## Connector

Connector是Tomcat的连接器，用于处理客户端请求。

### Connector的种类
一个Service可能包含多个Connector, 因为Tomcat支持多种协议，比如HTTP、HTTPS、AJP等。
Tomcat中常用的连接器有四种: 
- HTTP协议连接器 
- HTTPS协议连接器 
- AJP协议连接器 
- 其他协议连接器

### Connector的工作原理

Connector的工作原理是：客户端发送请求到Connector，Connector将请求传递给Container，Container处理请求并返回响应给Connector，Connector将响应返回给客户端。

### Connector结构

![connector_framwork](/resource/connector_framework.png)

Connector的核心是协议处理器(ProtocolHandler)，协议处理器负责解析请求和响应，处理请求和响应，将请求和响应传递给Container。

ProtocolHandler包括三个部分：
- 网络连接器（Endpoint）：负责建立和维护与客户端的连接。
- 处理器（Processor）：负责解析请求和响应，处理请求和响应。
- 适配器（Adapter）：负责将请求和响应传递给Container。

Endpoint处理的是网络连接(TCP/IP)，Processor处理的是请求和响应(HTTP/HTTPS/...)，Adapter处理的是Container。


## Container

Container是Tomcat的容器，用于管理Servlet的生命周期。
Container包括Engine、Host、Context和Wrapper。

### Container的结构

- Engine: 引擎，用于处理Connector传递过来的请求, 一个Service可以包含一个Engine;
- Host: 主机，用于处理Engine传递过来的请求, 一个Engine可以包含多个Host;
- Context: 上下文，用于处理Host传递过来的请求, 一个Host可以包含多个Context;
- Wrapper: 包装器，封装Servlet, 用于处理Context传递过来的请求, 一个Context可以包含多个Wrapper。

> Container是一个更加通用的概念，它是一个组件，用于管理Servlet的生命周期。Container包括Engine、Host、Context和Wrapper。

### Container的工作原理

Container处理请求使用的是责任链模式，请求从Engine开始，依次传递给Host、Context和Wrapper，最终由Wrapper处理请求。
