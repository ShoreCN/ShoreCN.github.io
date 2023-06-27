---
title: 公网ssh访问局域网内部Mac主机
date: 2023-01-04 10:01:00
categories:
- 工具
tags:
- ssh
- macOS
---

# 需求
我们的日常网络应用都是基于公网IP进行的，那么如果电脑处在于某个局域网内（例如公司网络、校园网络等情况），那么就常常出现使用公网IP无法访问的情况。要解决此问题涉及如下两部分配置：
- Mac端启动sshd服务
- 配置端口映射用以内访问内网设备

--------

# 1.Mac端启动sshd服务

系统版本：macOS High Sierra 10.13.1 (17B1003)
macOS系统自带了sshd服务，只需要启动即可，使用如下命令：
```
# 启动sshd服务
sudo launchctl load -w /System/Library/LaunchDaemons/ssh.plist
```
```
# 停止sshd服务
sudo launchctl unload -w /System/Library/LaunchDaemons/ssh.plist
```
```
# 查看sshd服务情况
sudo launchctl list | grep ssh
```

> 注意事项：
> 此处需要格外注意，必须要使用具有root权限的用户启动sshd服务，否则可能会因无法访问ssh key导致客户端ssh连接失败
> ssh_exchange_identification: Connection closed by remote host

> 参考资料
> https://www.cnblogs.com/EasonJim/p/7173859.html

# 2.端口映射外网访问局域网内部主机

位于公司、校园等局域网环境下面的多台主机，共用的都是一个公网IP，
所以外网直接通过此公网IP+端口访问某个局域网主机必然会产生混乱，此时就需要配置端口映射。

此处访问的是ssh服务，端口号为22，无法直接访问。
那么我们可以在局域网出口路由器上定义一个未使用的端口号（例如38998），映射到内网主机的端口22，具体如下：
外网终端 <--> 局域网出口路由器38998端口 <--> 内网主机22端口

------------

如此配置完之后，在外网就可以通过自定义的端口号和公网IP就可以访问内网的主机了

```ssh user_name@server_ip -p 38998```

如果需要通过ssh拷贝文件的话，可以采用scp命令

```scp -P 38998 src_file user_name@server_ip:dst_file```