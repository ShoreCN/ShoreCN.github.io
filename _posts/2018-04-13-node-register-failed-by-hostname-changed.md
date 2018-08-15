---
title: Kubernetes集群节点主机名称变化导致无法注册 
date: 2018-04-13 00:23:23
categories:
- 容器
tags: 
- Kubernetes
---

## 现象及原因
使用阿里云主机通常会通过命令hostname -b <name>修改主机名称便于后续使用和维护，但是在主机重启之后，通常又会被恢复成初始分配的默认名称。  
作为Kubernetes集群节点的情况下，这样的名称会变化的主机往往会导致已注册的节点重启后无法注册，通过kubelet日志可以看出，报错原因为主机当前名称和原始名称不同，导致节点无法注册到API server。
```
7月 04 09:46:50 k8scluster2master kubelet[1573]: I0704 09:46:50.750118    1573 kubelet_node_status.go:82] Attempting to register node izj6cgkv35o1qg19b4t96iz
7月 04 09:46:50 k8scluster2master kubelet[1573]: E0704 09:46:50.751935    1573 kubelet_node_status.go:106] Unable to register node "izj6cgkv35o1qg19b4t96iz" with API server: nodes "izj6cgkv35o1qg19b4t96iz" is forbidden: node "k8scluster2master" cannot modify node "izj6cgkv35o1qg19b4t96iz"
```


## 解决思路
在节点加入集群的动作中，因为默认会使用当前主机名称作为节点名称，所以节点注册之后如果主机名称发生变化会导致注册失败。那我们如果能指定主机每次注册的时候不使用当前主机名称，而是都使用相同的名称，就可以解决这类问题。  

## Kubelet
基于之前的思路到底是否可行，在这里我们先了解一下主机注册到Kubernetes集群中的原理以及其中的重要组件Kubelet。  
Kubelet组件运行在Node节点上，维持运行中的Pods以及提供kuberntes运行时环境，主要完成以下使命： 
1. 监视分配给该Node节点的pods 
2. 挂载pod所需要的volumes 
3. 下载pod的secret 
4. 通过docker/rkt来运行pod中的容器 
5. 周期的执行pod中为容器定义的liveness探针 
6. 上报pod的状态给系统的其他组件 
7. 上报Node的状态   

其中有一个很关键的就是Kubelet负责上报节点的信息到API server，那么按照我们预想的解决办法，若能控制这部分上报的信息就可以做到无论主机名称怎么变化，注册的节点名称还是不变的。

## 查看Kubelet配置
先查看一下当前机器的kubelet配置文件，可以通过下面这条名称获取，其中/etc/systemd/system/kubelet.service.d/10-kubeadm.conf就是配置文件了。
```
[root@k8scluster2master ~]# service kubelet status
Redirecting to /bin/systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since 三 2018-07-04 10:09:44 CST; 6min ago
     Docs: http://kubernetes.io/docs/
 Main PID: 20871 (kubelet)
   CGroup: /system.slice/kubelet.service
           └─20871 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --pod-manifest-path=/et...

```

让我们查看一下配置文件中的具体内容，可以看到这里指定了很多参数用以控制kubelet的运行状态。
```
[root@k8scluster2master ~]# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

查阅相关资料，可以找到一个参数就是符合我们的预想，用来指定主机注册时的节点名称的：—hostname-override  
那么我们就指定当前主机注册时的节点名称为k8scluster2master，试试看效果如何。
```
[root@k8scluster2master ~]# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_NODE_NAME_ARGS=--hostname-override=k8scluster2master"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS $KUBELET_NODE_NAME_ARGS
```

## 重新加载配置
执行如下命令使新增参数生效并检查新增参数是否已经生效
```
执行如下命令使新增参数生效
# systemctl stop kubelet
# systemctl daemon-reload
# systemctl start kubelet

检查新增参数是否已经生效
# ps -ef | grep kubelet
```

## 节点成功注册
修改完配置并重新加载之后，让主机再次加入集群，一会之后可以看到节点已经成功注册，大功告成。
```
[root@k8scluster2master ~]# kubectl get node
NAME                STATUS    ROLES     AGE       VERSION
k8scluster2master   Ready     master    74d       v1.10.1
k8scluster2node1    Ready     <none>    74d       v1.10.1
k8scluster2node2    Ready     <none>    74d       v1.10.1
```

## 参考资料
> [Kubelet组件解析 - CSDN博客](https://blog.csdn.net/jettery/article/details/78891733)  
