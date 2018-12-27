---
title: 使用NGINX Ingress进行Kubernetes集群负载均衡
date: 2018-12-25 00:23:23
categorys: Kubernetes
tags: Kubernetes
---

## 1 概述
Nginx Ingress Controller 是 Kubernetes Ingress Controller 的一种实现，作为反向代理将外部流量导入集群内部，实现将 Kubernetes 内部的 Service 暴露给外部，这样我们就能通过公网或内网直接访问集群内部的服务。

## 2 安装
Ingress的安装方式有两种
- 通过Kubernetes配置文件安装  
Ingress相关的yaml文件归档在如下地址：[kubernetes-ingress/deployments at master · nginxinc/kubernetes-ingress · GitHub](https://github.com/nginxinc/kubernetes-ingress/tree/master/deployments)
可以通过下载相关文件后使用kubectl apply直接进行部署，参阅指导文档[kubernetes-ingress/installation.md at master · nginxinc/kubernetes-ingress · GitHub](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md)，此处不予赘述

- 通过Helm安装  
本文主要介绍通过Helm安装使用Ingress，所以开始前请确保已经安装好Helm了。

### 2.1 选择服务暴露方式
Ingress对外部提供服务通常选择如下两种方式：
- LoadBalancer  
需要云厂商支持并购买，为每个LoadBalancer类型的Service分配公网IP地址

- hostPort  
hostPort是直接将容器的端口与所调度的节点上的端口连接起来，这样用户就可以通过宿主机的IP加上端口号来访问Pod了，如:  

```
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  containers:
    - name: influxdb
      image: influxdb
      ports:
        - containerPort: 8086
          hostPort: 8086
```
> hostPort的方案有个缺点，因为Pod重新调度的时候该Pod被调度到的宿主机可能会变动，这样就变化后用户必须自己维护一个Pod与所在宿主机的对应关系。

### 2.2 使用LoadBalancer提供服务
这种方式部署起来十分简单，因为stable/nginx-ingress这个helm包默认就是使用这种方式部署。确保在云厂商上已经开通了LoadBalancer服务之后，执行如下命令进行一键安装：  
`helm install --name nginx-ingress --namespace kube-system stable/nginx-ingress`

部署完成后查看结果：
```
$ kubectl get svc -n kube-system
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.3.255.138   119.28.121.125   80:30113/TCP,443:32564/TCP   21h
```


### 2.3 使用hostPort提供服务
之前有提到hostPort的缺点在于Pod被调度变化到其他宿主机时需要手动维护。那么我们通过DaemonSet将Nginx 的 Ingress Controller Pod指定到特定宿主机上面来解决这个问题，牺牲了一定的可靠性，但就可以不再操心如何手工维护服务的地址变化问题了，二者的利弊取舍可以根据使用场景进行评估。

#### 2.3.1 选择边缘节点
边缘节点：监听外部流量进入集群内部的节点
这里我们选择集群内部的一个节点作为边缘节点，注意这个节点需要有外部可以访问的公网地址并且80、443端口没有被其他应用占用。
```
[root@k8s-cluster1-master ~]# kubectl get nodes
NAME                      STATUS   ROLES    AGE   VERSION
izj6cfiv0zhc0j2k7ruyhsz   Ready    <none>   16d   v1.13.0
k8s-cluster1-master       Ready    master   16d   v1.13.0
```
以这个集群为例，我们选择izj6cfiv0zhc0j2k7ruyhsz作为边缘节点

#### 2.3.2 为边缘节点添加标签
我们使用如下指令给这个节点加上一个内容为node:edge的label，后续在DaemonSet部署时根据label进行绑定即可。
```
$ kubectl label node izj6cfiv0zhc0j2k7ruyhsz node=edge
node "izj6cfiv0zhc0j2k7ruyhsz" labeled
```

删除标签可以使用如下命令
```
$ kubectl label node izj6cfiv0zhc0j2k7ruyhsz node-
node "izj6cfiv0zhc0j2k7ruyhsz" labeled
```


#### 2.3.3 使用helm安装ingress
配置好边缘节点之后即可以使用Helm安装ingress了，安装命令如下：
```
helm install stable/nginx-ingress \
  --namespace kube-system \
  --name nginx-ingress \
  --set controller.kind=DaemonSet \
  --set controller.daemonset.useHostPort=true \
  --set controller.nodeSelector.node=edge \
  --set controller.service.type=ClusterIP
```

参数解释：
- namespace  可选项，配置后可指定ingress安装的命令空间，默认为default
- name  生成的资源名称
- set controller.kind=DaemonSet  选择部署方式为DaemonSet
- set controller.daemonset.useHostPort=true  开启hostPort功能，可以直接通过节点访问ingress服务
- controller.nodeSelector.node=edge  提供对外提供服务的节点筛选方式，需要和之前配置的节点label匹配，例如之前配置的node=edge，这里也需要填上一样的键值
- set controller.service.type=ClusterIP  内部服务的类型

安装完成之后检查一下对应pod是否都正常运行了
```
[root@k8s-cluster1-master ~]# kubectl get pods -n kube-system | grep nginx-ingress
nginx-ingress-controller-hv8h9                   1/1     Running   0          15d
nginx-ingress-default-backend-56d99b86fb-4jlxn   1/1     Running   0          15d
```

## 3 配置
### 3.1 创建测试服务
Ingress安装完成之后，就可以配置相应规则来对外暴露服务了。我们可以创建一个服务来测试
创建一个my-nginx.yaml，内容为
```
  apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    run: my-nginx
```

通过`kubectl apply -f my-nginx.yaml`进行创建，通过`kubectl get service`可以看到my-nginx服务已经创建成功
```
[root@k8s-cluster1-master ~]# kubectl get svc
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes                           ClusterIP   10.96.0.1        <none>        443/TCP          17d
my-nginx                             ClusterIP   10.100.181.157   <none>        80/TCP,443/TCP   13d
```


### 3.2 创建ingress规则
创建完成服务之后，我们就可以通过配置ingress规则来暴露服务了，
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: www.10cloud.tk
    http:
      paths:
      - path: /web
        backend:
          serviceName: my-nginx
          servicePort: 80
  tls:
  - hosts:
    - www.10cloud.tk
    secretName: www-10cloud-tk-tls
```

参数解释
- kubernetes.io/ingress.class: "nginx"  
通过定义kubernetes.io/ingress.class 这个annotation可以在有多个ingress controller的情况下能够让请求被我们安装的这个处理
- nginx.ingress.kubernetes.io/rewrite-target: /  
配置位置重定向，作用是使Ingress以根路径转发到后端，避免访问路径错误配置而导致的404错误。参数详细说明参照[ingress-nginx/docs/examples/rewrite at master · kubernetes/ingress-nginx · GitHub](https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/rewrite)
- host  
域名地址
- path  
外部访问的相对地址
- backend  
通过配置服务名称和服务端口来指定和path绑定的后端服务
- tls  
创建了证书之后，可以通过tls绑定已创建的secret来实现https安全访问功能

至此，我们的Ingress创建完成并且通过配置服务与域名的关系，实现了外部访问的功能，通过域名`chenshuo.tk/web`即看到nginx的内容，说明通过域名已经可以访问集群内的服务了
```
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```

