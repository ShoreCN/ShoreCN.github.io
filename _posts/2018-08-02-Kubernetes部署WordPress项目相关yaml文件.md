---
title: Kubernetes部署WordPress项目相关yaml文件
date: 2018-08-02 15:31:30
categories:
- 容器
tags: Kubernetes
---

# Kubernetes部署WordPress项目相关yaml文件
## MySQL
### 部署
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels: &id001
    app: wordpress-demo-mysql
    app_id: '262'
    internal_name: wordpress-demo-mysql.wordpress-demo-mysql
  name: wordpress-demo-mysql.wordpress-demo-mysql
spec:
  replicas: 1
  selector:
    matchLabels: *id001
  template:
    metadata:
      labels: *id001
    spec:
      containers:
      - image: 47.75.159.100:5000/wordpress-demo/wordpress-demo-mysql-pure:5.7
        name: mysql
        ports:
        - containerPort: 3306
          name: mysql
          protocol: TCP
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: sqsm1234
```

### 服务
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app_id: '261'
    internal_name: wordpress-demo.wordpress-demo-mysql
  name: wordpress-demo-mysql
spec:
  ports:
  - name: mysql
    port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: wordpress-demo-mysql
    app_id: '262'
  type: ClusterIP
```


## WordPress
### 部署
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels: &id001
    app: wordpress-demo-web
    app_id: '266'
    internal_name: wordpress-demo-web.wordpress-demo-web
  name: wordpress-demo-web.wordpress-demo-web
spec:
  replicas: 1
  selector:
    matchLabels: *id001
  template:
    metadata:
      labels: *id001
    spec:
      containers:
      - image: 47.75.159.100:5000/wordpress-web-apche/wordpress-web-apche:4.9.7
        name: web
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-demo-mysql
        - name: WORDPRESS_DB_PASSWORD
          value: sqsm1234
```

### 服务
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app_id: '261'
    internal_name: wordpress-demo.wordpress-demo-web
  name: wordpress-demo-web
spec:
  ports:
  - name: http
    port: 6767
    protocol: TCP
    targetPort: 80
  selector:
    app: wordpress-demo-web
    app_id: '266'
  type: ClusterIP
```
