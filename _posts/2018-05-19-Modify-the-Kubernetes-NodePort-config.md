---
title: 修改Kubernetes的NodePort范围
date: 2018-05-19 22:46:49
categories:
- 容器
tags: Kubernetes
---

## 问题
在我们的实际使用过程中，如果想让服务能从外部访问，就会使用到NodePort服务类型。默认的选择了NodePort类型之后，端口是从30000-32767范围内随机分配，就算用户自行指定其他端口号，也会报如下错误：
```
The Service "test" is invalid: spec.ports[0].nodePort: Invalid value: 7712: provided port is not in the valid range. The range of valid ports is 30000-32767
```

而我们实际的应用场景中，肯定有需要指定端口号的情况，例如服务要绑定域名只能用默认的80端口，而Kubernetes这样的不灵活限制了我们的使用。好在只需要修改API Server一个配置就可以调整NodePort的范围。

## API Server简介
API Server提供了Kubernetes各类资源对象的增删查改等HTTP Rest接口，是整个系统的数据总线和数据中心。

kubernetes API Server的功能：
- 提供了集群管理的REST API接口(包括认证授权、数据校验以及集群状态变更)；
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）;
- 是资源配额控制的入口；
- 拥有完备的集群安全机制.

## API Server启动参数说明
下面是一些典型的API Server参数说明，其中service-node-port-range就是我们需要关注修改的配置字段了。
```
--etcd-servers：指定etcd服务的URL
--insecure-bind-address：apiserver绑定主机的非安全IP地址，设置0.0.0.0表示监听所有IP
--insecure-port：apiserver绑定主机的非安全端口号，默认为8080
--service-cluster-ip-range：Kubernetes集群中Service的虚拟IP地址段范围，以CIDR格式表示，该IP范围不能与物理机的真实IP段有重合
--service-node-port-range：Kubernetes集群中Service可映射的物理机端口范围，默认为30000-32767
--admission-control：Kubernetes集群的准入控制设置爱，个控制模块以插件的形式生效
```

## 修改方案
修改API Server的配置，配置文件路径/etc/kubernetes/manifests/kube-apiserver.yaml，新增一行--service-node-port-range=80-32767
```
spec:
  containers:
  - command:
    - kube-apiserver
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --secure-port=6443
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --requestheader-username-headers=X-Remote-User
    - --requestheader-group-headers=X-Remote-Group
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --insecure-port=0
    - --enable-bootstrap-token-auth=true
    - --allow-privileged=true
    - --requestheader-allowed-names=front-proxy-client
    - --advertise-address=172.31.59.193
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --authorization-mode=Node,RBAC
    - --etcd-servers=https://127.0.0.1:2379
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --service-node-port-range=80-32767
```

配置修改之后Kubernetes会自动重新加载生效，不需要我们再做其他操作了，现在新创建的服务可以使用80端口对外通信了。

> 参考资料  
> [二、安装并配置Kubernetes Master节点 - Federico - 博客园](https://www.cnblogs.com/Cherry-Linux/p/7841273.html)  

#Kubernetes#
