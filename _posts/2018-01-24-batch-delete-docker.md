---
title: 批量删除docker内已停止的容器与镜像
date: 2018-01-24 23:30:04
categories:
- Kubernetes
---

# 批量删除docker内已停止的容器与镜像
	Docker的使用过程中，常常出现大量已经退出的或启动失败的容器，这时候如果要逐个删除的话费时费力，好在还有如下几种方法进行批量删除，节省时间大大提高工作效率。

## 方法一
显示所有的容器，过滤出Exited状态的容器，取出这些容器的ID，
`sudo docker ps -a|grep Exited|awk '{print $1}'`

查询所有的容器，过滤出Exited状态的容器，列出容器ID，删除这些容器
```
sudo docker rm `docker ps -a|grep Exited|awk '{print $1}'`
```

## 方法二
删除所有未运行的容器（已经运行的删除不了，未运行的就一起被删除了）
`sudo docker rm $(sudo docker ps -a -q)`

## 方法三
根据容器的状态，删除Exited状态的容器
`sudo docker rm $(sudo docker ps -qf status=exited)`
 
## 方法四
Docker 1.13版本以后，可以使用 docker containers prune 命令，删除孤立的容器。
`sudo docker container prune`

## 删除所有镜像
`sudo docker rmi $(docker images -q)`
