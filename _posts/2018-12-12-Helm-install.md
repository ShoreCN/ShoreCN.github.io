---
title: Linux系统下Helm（Kubernetes包管理器）的安装使用
date: 2018-12-11 11:14:39
categories:
- Kubernetes
tags:
- Kubernetes
---

## 概述
Helm 是 Kubernetes 的包管理器，可以帮我们简化 kubernetes 的操作，一键部署应用。假如你的机器上已经安装了 kubectl 并且能够操作集群，那么你就可以安装 Helm 了。

## 安装
### 安装客户端（Helm）
官方提供了多种安装方式，这里选用脚本安装的方法，使用命令`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash`会自动抓取最新的版本完成客户端的安装：
```
[root@k8s-cluster1-master ~]# curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  7236  100  7236    0     0  74494      0 --:--:-- --:--:-- --:--:-- 74597
Downloading https://kubernetes-helm.storage.googleapis.com/helm-v2.12.0-linux-amd64.tar.gz
Preparing to install helm and tiller into /usr/local/bin
helm installed into /usr/local/bin/helm
tiller installed into /usr/local/bin/tiller
Run 'helm init' to configure helm.
```


### 安装服务端（Tiller）
Helm安装完成之后，通过命令行`helm init`即可完成服务端Tiller的安装。
```
[root@k8s-cluster1-master ~]# helm init
Creating /root/.helm
Creating /root/.helm/repository
Creating /root/.helm/repository/cache
Creating /root/.helm/repository/local
Creating /root/.helm/plugins
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

使用命令`helm version`可以查看安装是否成功和版本号情况
```
[root@k8s-cluster1-master ~]# helm version
Client: &version.Version{SemVer:"v2.12.0", GitCommit:"d325d2a9c179b33af1a024cdb5a4472b6288016a", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.12.0", GitCommit:"d325d2a9c179b33af1a024cdb5a4472b6288016a", GitTreeState:"clean"}
```

### 权限配置
默认安装的 tiller 权限很小，我们执行下面的脚本给它加最大权限，这样方便我们可以用 helm 部署应用到任意 namespace 下:
```
kubectl create serviceaccount --namespace=kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl patch deploy --namespace=kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
