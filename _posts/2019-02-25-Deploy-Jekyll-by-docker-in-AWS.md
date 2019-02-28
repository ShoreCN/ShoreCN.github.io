---
title: 在AWS Ubuntu服务器上使用Docker快速部署Jekyll静态博客
date: 2019-02-25 22:46:49
categories:
- 工具
---

## Jekyll简介
Jekyll 是一个简单且完全免费的静态博客站点生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个只用 Markdown (或 Textile)、Liquid、HTML & CSS 就可以生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。

## Docker简介
Docker是一个开放源代码软件项目，让应用程序部署的工作可以自动化进行，借此在Linux操作系统上，提供一个额外的软件抽象层，以及操作系统层虚拟化的自动管理机制。

## Jekyll部署
### 环境
AWS Ubuntu 16.04.5 LTS

### 安装
#### Docker安装
Docker的安装资料已经十分丰富了，参见官网指导即可。
> [Get Docker CE for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)  

#### Jekyll安装
因为使用Docker，所以免去了Jekyll安装。不用再关心系统版本、依赖组件之类的问题，开箱即用。

### 选择Jekyll模板
网上有大量的现成Jekyll模板可以选择，此处以Jekyll-Now为例。
在服务器上创建一个目录并git clone模板代码：
```
mkdir jekyllBlog
cd jekyllBlog
git clone https://github.com/barryclark/jekyll-now.git .
```

### 构建
Jekyll提供了一条build命令将模板编译为静态网页，直接在执行如下Docker命令即可：
```
export JEKYLL_VERSION=3.8
docker run --rm --volume="$PWD:/srv/jekyll" -it jekyll/jekyll:$JEKYLL_VERSION jekyll build
```

可以看到新生成一个_site目录，就是刚刚构建出来的网页文件

### 开启服务
Jekyll 同时也集成了一个网页服务，通过server命令即可开启。
`docker run --rm --volume="$PWD:/srv/jekyll" -it -p 4000:4000 jekyll/jekyll:$JEKYLL_VERSION jekyll server`

通过-p参数指定了对外暴露的的服务器端口为4000

至此，我们就可以通过服务器的IP加端口（例如http://1.2.3.4:4000）访问刚刚创建好的静态博客系统了。
![jekyllstartuppage](https://raw.githubusercontent.com/ShoreCN/ShoreCN.github.io/master/resource/jekyllStartupPage.png)


> 参考资料  
> [Setup Jekyll via Docker? - Stack Overflow](https://stackoverflow.com/questions/53470794/setup-jekyll-via-docker)  
