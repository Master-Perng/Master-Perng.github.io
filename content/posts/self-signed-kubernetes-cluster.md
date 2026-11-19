---
title: "自签名集群搭建"
date: "2022-12-13T16:44:36+08:00"
previewImage: "/images/covers/cloud-native.svg"
draft: false
categories:
  - "云原生"
tags:
  - "Kubernetes"
  - "TLS"
  - "证书"
---
## 1. 生成2048位的ca.key

~~~bash
openssl genrsa -out ca.key 2048
~~~

## 2. 在 ca.key 文件的基础上，生成 ca.crt 文件

~~~bash
openssl req -x509 -new -nodes -key ca.key -subj "/CN=175.27.155.72" -days 3650 -out ca.crt
~~~

## 3. 生成apiserver.key

~~~bash
openssl genrsa -out apiserver.key 2048
~~~

## 4. 创建一个用于生成证书签名请求（CSR）的配置文件

~~~bash
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = CN
ST = Beijing
L = Beijing
O = Kubernetes
OU = Kubernetes
CN = master.perng.cn

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
DNS.5 = master.perng.cn
IP.1 = 175.27.155.72
IP.2 = 10.0.4.4
IP.3 = 172.17.0.1

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
~~~

~~~bash
openssl req -new -key apiserver.key -out apiserver.csr -config csr.conf
~~~

## 5. 基于ca 、apiserver.csr等文件生成服务端证书

~~~bash
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out apiserver.crt -days 3650 \
    -extensions v3_ext -extfile csr.conf -sha256
~~~

## 6. 初始化集群

​     后续过程跟正常搭建集群一样。

