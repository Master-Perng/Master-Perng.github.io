---
title: "利用 CoreDNS 枚举集群Service & Port"
date: "2022-11-23T11:13:46+08:00"
previewImage: "/images/covers/cloud-native.svg"
draft: false
categories:
  - "云原生"
tags:
  - "Kubernetes"
  - "CoreDNS"
  - "DNS"
  - "内网安全"
---
# 利用 CoreDNS 枚举集群Service & Port

## 1. 发现历程

看代码的过程中，发现在/pkg/dnsutil/reverse.go存在 关于**in-addr.arpa**的定义

![image-20221111162519248](/images/legacy/image-20221111162519248.png)

跟了一下，走到 /plugin/hosts/hosts.go ，发现CoreDNS是支持PTR解析记录的

![image-20221111163058551](/images/legacy/image-20221111163058551.png)

看下**k8s_external**包，这个包是处理svc域名的一些操作。109行发现K8s插件也是支持ptr记录的，CoreDNS的使用过程是集群内部的域名，coreDNS分发至kubedns，集群外的域名则通过coreDNS进行递归。

## 2. 利用手法

在公共DNS服务器上，很少会有ip存在PTR解析记录，而集群内部所有的域名都是通过Service进行管理，这样也就说明，如果CoreDNS的K8s插件若支持PTR记录，那么集群内部就可以通过ip来获取svc域名

![image-20221111165318548](/images/legacy/image-20221111165318548.png)

正常情况下，查询一个service的A记录是这样的

![image-20221111165553003](/images/legacy/image-20221111165553003.png)

~~~
<SVC>.<NAMESPACE>.svc.cluster.local
~~~

若PTR可用，我们将会得到重要的三个信息，一个是命名空间、一个是svc名称、一个就是svcip，事实证明，这条路可行。

![image-20221111165827487](/images/legacy/image-20221111165827487.png)

那么按照这个思路，就可以通过PTR记录进行所有的Service的枚举了。

## 3. 实现

在集群容器内部，由于需要使用CoreDNS，/etc/resolv.conf文件会与普通的机器不同

![image-20221111170027212](/images/legacy/image-20221111170027212.png)

![image-20221111170055175](/images/legacy/image-20221111170055175.png)

由于K8s配置过程中需要将CoreDNS与svc放在同一个网段，这样一来，通过容器内部的**/etc/resolv.conf**文件就可以知道集群svc所在ip段，通过Fuzz这个段所有的ip将会得到完整的svc情况。

![image-20221111170342760](/images/legacy/image-20221111170342760.png)

根据这个原理，写出了个脚本，跑所有svc地址。

![image-20221111170444998](/images/legacy/image-20221111170444998.png)

一个B段6000并发下仅需要几秒。

![image-20221111170737785](/images/legacy/image-20221111170737785.png)

## 4. 端口探测

以上的所有操作是将所有的svc名字、命名空间以及ip地址枚举了出来，但是在svc当中还有一个重要信息，那就是**端口**，其实按照上文描述的方法已经可以对存活svc进行端口扫描了，但是就这么做的话，不优雅，因此再次打开CoreDNS的代码，找找优雅的办法。

/plugin/k8s_external/external.go 107行

![image-20221115105245941](/images/legacy/image-20221115105245941.png)

这是允许通过的四种dns类型，A为 域名-ip4 AAAA为 域名-ip6 PTR为 ip-域名 SRV为 服务-端口

前三种已经都被我们所利用了，唯独第四种SRV记录还没被使用。

**SRV记录**：

英语：**Service Record**，中文又名**服务定位记录**是域名系统中用于指定服务器提供服务的位置（如主机名和端口）数据。此数据于RFC 2782中定义，类型代码为33。部分协议，如会话发起协议（SIP）及可扩展消息与存在协议（XMPP）通常需要服务记录的支持。

正常使用是这样的

![image-20221115105654799](/images/legacy/image-20221115105654799.png)

~~~sh
nslookup -qt=srv _<service>._<protocol>.domain.com
~~~

在集群当中测试下

![image-20221115105923277](/images/legacy/image-20221115105923277.png)

按照这个写法不行，但是代码中写了，说明肯定是支持的，把svc、命名空间、协议进行排列组合，看哪一种是可行的。最终发现，集群内的srv域名查询需要符合以下格式。

~~~bash
nslookup -qt=srv <svc>.<namespace>.svc.cluster.local
~~~



![image-20221115110248779](/images/legacy/image-20221115110248779.png)

其中30111端口的，即为svc开放端口

![image-20221115110356697](/images/legacy/image-20221115110356697.png)

将这个东西融入之前的脚本，达到如下效果。

![image-20221115110659114](/images/legacy/image-20221115110659114.png)

~~~json
{"default":[{"SVC":"kubernetes","IP":"10.1.0.1","Port":[443]}],"kube-system":[{"SVC":"kube-dns","IP":"10.1.0.10","Port":[9153,53]}],"moresec-system":[{"SVC":"ksp-fileserver","IP":"10.1.3.195","Port":[7777]},{"SVC":"ksp-redis-master","IP":"10.1.8.79","Port":[6379]},{"SVC":"gatekeeper-webhook-service","IP":"10.1.12.127","Port":[443]},{"SVC":"ksp-mongodb","IP":"10.1.53.252","Port":[27017]},{"SVC":"ksp-cluster-manager","IP":"10.1.1.43","Port":[5020]},{"SVC":"ksp-core","IP":"10.1.59.129","Port":[8010,5006]},{"SVC":"ksp-admission","IP":"10.1.56.115","Port":[8090]},{"SVC":"ksp-license","IP":"10.1.72.216","Port":[5030]},{"SVC":"ksp-cluster-manager-proxy","IP":"10.1.88.58","Port":[8080]},{"SVC":"ksp-minio","IP":"10.1.92.117","Port":[9001,9000]},{"SVC":"ksp-webapi-srv","IP":"10.1.95.161","Port":[5002]},{"SVC":"ksp-scanner","IP":"10.1.139.11","Port":[5103]},{"SVC":"ksp-gateway-proxy","IP":"10.1.213.112","Port":[8080]},{"SVC":"ksp-controller","IP":"10.1.222.32","Port":[5040]},{"SVC":"ksp-web","IP":"10.1.245.240","Port":[443]},{"SVC":"ksp-nsqd","IP":"10.1.255.210","Port":[4151,4150]}],"work":[{"SVC":"tomcat8","IP":"10.1.229.92","Port":[80]}],"zhuri":[{"SVC":"tp-svc","IP":"10.1.0.47","Port":[15443]},{"SVC":"tomcat-svc","IP":"10.1.34.246","Port":[9080]},{"SVC":"baota-svc","IP":"10.1.47.228","Port":[18800,18080,18888]},{"SVC":"sqli-svc","IP":"10.1.224.165","Port":[18081]}]}
~~~

但是值得一提的是，这个方法拉出的port不区分tcp udp，但是我使用过的几个集群来讲，除了CoreDNS以外，很少会有udp端口的出现。

## 5. 解决办法

对于业务不需要srv与ptr记录的集群，禁用coreDNS的ptr与srv记录

~~~bash
#修改CoreDNS的配置文件
kubectl edit configmap/coredns -n kube-system
~~~

加入如下两句配置

![image-20221123111346008](/images/legacy/image-20221123111346008.png)

经过该配置，CoreDNS会拦截掉ptr与srv记录。

![image-20221123111639583](/images/legacy/image-20221123111639583.png)
