---
title: Elements
date: 2018-07-11 23:29:08
categories:
- 工具
tags:
- GitHub
---

## GitHub OAuth第三方登录
## 前言
	OAuth分1.0、1.0A、2.0三个版本，前二者已经淘汰以下所介绍的都是指2.0版本。
	本文将以第三方网站获取GitHub代码仓库为例，简述OAuth的运作流程。

## OAuth介绍
> 	开放授权（OAuth）是一个开放标准，允许用户让第三方应用访问该用户在某一网站上存储的私密的资源（如照片，视频，联系人列表），而无需将用户名和密码提供给第三方应用  
>   
> 	OAuth允许用户提供一个令牌，而不是用户名和密码来访问他们存放在特定服务提供者的数据。每一个令牌授权一个特定的网站（例如，视频编辑网站)在特定的时段（例如，接下来的2小时内）内访问特定的资源（例如仅仅是某一相册中的视频）。这样，OAuth让用户可以授权第三方网站访问他们存储在另外服务提供者的某些特定信息，而非所有内容。  

> https://zh.wikipedia.org/wiki/开放授权  
> ——维基百科  

## OAuth原理
![](https://github.com/ShoreCN/ShoreCN.github.io/blob/master/resource/1-OAuth2-Authorization-Code-Flow.png?raw=true)
	我们把OAuth2的整个认证过程大致分为三个阶段。第一阶段主要是向用户取得授权许可，对应图中的第1、2、3步；第二阶段主要是申请访问令牌（access_token），对应图中的第4、5步；第三阶段就是使用access_token获取用户数据。

> 参考资料： [移花接木：针对OAuth2的攻击 – ThoughtWorks洞见](http://insights.thoughtworkers.org/attack-aim-at-oauth2/)  

## OAuth应用实例
案例背景：
我们有这么一个应用——可以让用户绑定GitHub账号，然后使用仓库代码在线构建容器镜像的平台。所以我们要获取用户的GitHub仓库信息用于后续下载代码、构建容器等流程，此处我们就可以使用GitHub OAuth达到这个目的。

#### 在GitHub上创建OAuth App
1. 打开 Setting > Developer setting > OAuth applications
2. 点击 Register a new application
3. 填入基本的app信息

#### 获取对应的Client ID与Client Secret
OAuth应用创建完成之后，可以在应用信息页面查看到Client ID与Client Secret的值，二者是完成OAuth认证流程的关键。

#### 应用生成获取GitHub授权的链接
然后就来到了我们的应用与GitHub联系的关键点，在应用上可能就是一个简单的绑定GitHub账号的按钮，本质上是一串由关键参数组成的URL，下面我们详细说一下这个URL是如何生成的。
```
URL样例：
https://github.com/login/oauth/authorize?client_id=5448811562b83dadc3cf&scope=repo%2Cuser%3Aemail&redirect_uri=http%3A%2F%2Fcd.10.com%2Fapi%2Fgithub%2Foauth%2Fcallback%3Fredirect_url%3Dhttp%253A%252F%252Fwww.baidu.com%26uid%3D99
```

乍看之下，这个URL杂乱无章无从下手，但其实知晓了原理之后就很好理解了。
这个URL承载了GItHub OAuth认证的所有步骤，那么回到之前的原理解析，其实这个URL就包含了两个动作：
1. 跳转到GitHub界面申请用户授权
https://github.com/login/oauth/authorize 
这是GitHub的认证API，传入的参数是client_id, scope, redirect_uri 
- client_id: 之前步骤中创建OAuth App的时候获取到的字段，传入此参数可以让GitHub知道到底是哪个App在申请用户的授权
- scope: 需要向用户申请授予的权限，具体规则可参考 ![GitHub API Doc](https://developer.github.com/apps/building-oauth-apps/scopes-for-oauth-apps/)
- redirect_uri: 如果用户完成授权后则跳转到此页面，在这个页面的请求中GitHub会带上code参数，用于下一步获取token

2. 申请访问令牌
在上一步拿到GitHub返回的code参数之后，我们就可以去获取具有访问GitHub权限的token了。
https://github.com/login/oauth/access_token
获取token的API，传入参数client_id, client_secret, code
- client_id: 之前步骤中创建OAuth App的时候获取到的字段，传入此参数可以让GitHub知道到底是哪个App在使用用户的授权
- client_secret: 之前步骤中创建OAuth App的时候获取到的字段，传入此参数可以让GitHub知道此次操作是否属于该App的合法使用者
- code: GitHub提供给App的合法认证标志

从token API的返回值中取出参数access_token，用于我们后续去获取GtiHub资源。

3. 获取GitHub资源
我们这里以获取用户下的仓库列表为例，演示如何使用OAuth token，。
https://api.github.com/user/repos
这是获取仓库的API，假若token值为123456789abc，那么在GET的请求头中加入`{‘Authorization’: ‘token 123456789abc’}`，就可以拿到GitHub数据了。
返回值：
- repos_url: 仓库地址
- repos_name： 仓库名称
- http_url ：仓库https地址

至此，我们就完成了使用OAuth授权、认证、访问资源的全流程，其他接口类似，只要有了token就有了权限可以完成所有资源获取的操作。

