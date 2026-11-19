---
title: "HTTPS四层协议代理服务器搭建"
date: "2022-02-04T01:24:30+08:00"
previewImage: "/images/covers/tech-notes.svg"
draft: false
categories:
  - "技术实践"
tags:
  - "代理"
  - "Squid"
  - "CONNECT"
  - "网络"
---
**HTTP代理在白皮书当中氛围两种，一种为RFC 7230的HTTP代理，以及RFC 7231 的 CONNECT 方法代理，前者只可以通过七层的HTTP的请求包，后者即为HTTPS代理，可通过CONNECT方法走四层协议包。**

## 0.1 利用组件  Squid

### 0.11 直接 yum 下载安装

~~~shell
yum install squid
~~~

### 0.12 配置文件

![image-20211231115731144](/images/legacy/20211231115732.png)

配置文件中存在acl的概念，将acl放开1-65535即全部端口放行，如果不需要全放行可根据自己需求进行使用，例如，搭建squid的目的为代理访问 22 端口，则开放acl 22端口，以此类推。

其次重要的是acl放行 CONNECT 方法，这是走四层包的核心功能。

![image-20211231120037649](/images/legacy/20211231120039.png)

修改 此字段，将不在ssl端口的connect方法放行。

![image-20211231120214960](/images/legacy/20211231120216.png)

修改此字段，改为允许。

可以通过四层包

![image-20211231161621981](/images/legacy/20211231161623.png)




