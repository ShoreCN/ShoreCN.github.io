---
title: CentOS部署Kubernetes集群
date: 2018-03-30 21:35:29
categories:
- 容器
tags: Kubernetes
---

# CentOS部署Kubernetes集群
## 相关组件版本

| 名称 | 版本号 |
|:---|:---|
|docker | 1.13.1|
|kubeadm | 1.10.1|
|kubectl| 1.9.3|

## 安装Docker
```
yum install -y docker
systemctl enable docker && systemctl start docker
```

## 设置节点名称
Kubernetes集群中的所有节点名称不能相同，所以在创建集群前先修改好各个节点的名称防止冲突。  
`hostname -b 名称`  

## 安装Kubernetes组件
### 配置yum镜像源文件
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

### 关闭防火墙
`setenforce 0`

### 安装组件
` yum install -y kubelet kubeadm kubectl `

### 启动服务
` systemctl enable kubelet && systemctl start kubelet `

最后通过命令`running sysctl net.bridge.bridge-nf-call-iptables=1`将/proc/sys/net/bridge/bridge-nf-call-iptables设置为1，否则后续Master创建集群和Node加入集群会产生错误。

## Master创建集群
### 选择CNI
Kubernetes集群的正常使用依赖于CNI(Container Network Interface)。
常见的CNI有如下几种：
- Calico
- Canal
- Flannel
- Kube-router
- Romana
- Weave Net
在这个部署场景中我们选用Flannel

### 初始化集群
`kubeadmin init --pod-network-cidr=10.244.0.0/16 `  
为了保证Flannel可以正常工作，需要在Kubeadm init时加上“—pod-network-cidr=10.244.0.0/16 ”参数，使用其余CNI对应的参数各不相同，详情请参考[官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)

若输出以下类似内容则代表集群已经初始化成功了：
```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.31.59.194:6443 --token 0vfzv1.vw6oxga0xhc1lzm5 --discovery-token-ca-cert-hash sha256:39b0319100e9997cb859ef6967a00c93de3ef8e535a7dc3d6e4ed510a5b97e6c
```

注意其中的kubeadm join内容，保存好以便后续Node节点加入集群使用。


### 配置环境参数
留意kubeadm init成功之后的回显，其中提醒了我们使用集群前需要做的相应配置。  
`mkdir -p $HOME/.kube`  
`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`  
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`  


### 安装CNI
我们使用Flannel作为这个集群的CNI，所以使用如下命令安装Flannel：  
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

其余类型的CNI的安装方式可以参照[官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)

## Node加入集群
现在可以将已经安装好Kubernetes组件的主机作为Node节点加入我们已经创建好的集群中了。
使用类似如下kubeadm join命令完成Node节点的加入（token来自于之前Master上kubeadm init的结果）：  
`kubeadm join 172.31.59.194:6443 --token 0vfzv1.vw6oxga0xhc1lzm5 --discovery-token-ca-cert-hash sha256:39b0319100e9997cb859ef6967a00c93de3ef8e535a7dc3d6e4ed510a5b97e6c`

## 查看集群状态
到此，我们的集群已经创建完成，可以在Master上通过如下命令查看集群成员：  
`kubectl get nodes`

结果类似如下格式，若各节点状态都已经为Ready则代表集群建立完成：
```
[root@izj6cfshug9pwgb71j3t10z ~]# kubectl get node
NAME                      STATUS    ROLES     AGE       VERSION
izj6cfshug9pwgb71j3t10z   Ready     master    58m       v1.10.1
k8scluster1node1          Ready     <none>    24m       v1.10.1
k8scluster1node2          Ready     <none>    1m        v1.10.1
```


