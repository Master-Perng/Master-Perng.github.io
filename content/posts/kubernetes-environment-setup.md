---
title: "Kubernetes 环境搭建"
date: "2022-10-13T15:26:36+08:00"
previewImage: "/images/covers/cloud-native.svg"
draft: false
categories:
  - "云原生"
tags:
  - "Kubernetes"
  - "Docker"
  - "集群搭建"
---
## 所有节点 绑定Hosts

~~~bash
# 不绑定会导致初始化失败
[root@Perng-Node2 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.159.100 Perng-Master 
192.168.159.110 Perng-Node1 
192.168.159.120 Perng-Node2
~~~



## 所有节点 安装Docker

~~~bash
#安装yum-utils
[root@Perng-Master ~]# yum install yum-utils
[root@Perng-Master ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#更新元数据
[root@Perng-Master ~]# yum makecache fast
#查找docker版本
[root@Perng-Master ~]# yum list docker-ce.x86_64 --showduplicates | sort -r
#安装合适版本的docker
[root@Perng-Master ~]# yum -y install docker-ce-20.10.17-3.el7 .centos
#加入启动项
[root@Perng-Master ~]# systemctl enable docker.service
#启动docker访问
[root@Perng-Master ~]# service docker start
~~~

## 所有节点 配置K8s环境

~~~bash
# 关闭防火墙
[root@Perng-Master ~]# systemctl disable firewalld
[root@Perng-Master ~]# systemctl stop firewalld
# 关闭selinux
[root@Perng-Master ~]# sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
[root@Perng-Master ~]# sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
# 禁用交换分区
[root@Perng-Master ~]# sed -i 's/.*swap.*/#&/' /etc/fstab
# 修改内核参数
[root@Perng-Master ~]# cat <<EOF >/etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF

[root@Perng-Master ~]#sysctl --system # sysctl -p
~~~

## Master节点 安装k8s组件

~~~bash
# 选个合适版本k8s
[root@Perng-Master ~]# yum list kubeadm --showduplicates | sort -r
[root@Perng-Master ~]# POD_NETWORK=10.244.0.0
# 配置k8s阿里云源
[root@Perng-Master ~]# cat <<EOF >/etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=Kubernetes
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
> enabled=1
> gpgcheck=0
> repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg     
> EOF
# 更新yum源
[root@Perng-Master ~]# yum clean all
[root@Perng-Master ~]# yum -y makecache
# 安装kubeadm、kubectl、kubelet
[root@Perng-Master ~]# yum install -y kubectl-1.22.3-0 kubeadm-1.22.3-0 kubelet-1.22.3-0
# 启动kubelet服务
[root@Perng-Master ~]# systemctl enable kubelet && systemctl start kubelet
~~~

## Node节点安装K8s组件

~~~bash
# 配置阿里K8s源
[root@Perng-Node1 ~]# cat <<EOF >/etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=Kubernetes
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
> enabled=1
> gpgcheck=0
> repo_gpgcheck=0
> gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg     
> EOF

# 更新yum源
[root@Perng-Node1 ~]# yum clean all
[root@Perng-Node1 ~]# yum -y makecache
# 安装kubeadm、kubectl、kubelet  版本建议与master节点相同，不相同不要超过两个版本
[root@Perng-Node1 ~]# yum install -y kubectl-1.22.3-0 kubeadm-1.22.3-0 kubelet-1.22.3-0
# 启动kubelet服务
[root@Perng-Node1 ~]# systemctl enable kubelet && systemctl start kubelet
~~~

## Master节点 拉取K8s镜像

~~~bash
# 查看自己安装k8s版本的组件版本
[root@Perng-Master ~]# kubeadm config images list --kubernetes-version=v1.22.3
k8s.gcr.io/kube-apiserver:v1.22.3
k8s.gcr.io/kube-controller-manager:v1.22.3
k8s.gcr.io/kube-scheduler:v1.22.3
k8s.gcr.io/kube-proxy:v1.22.3
k8s.gcr.io/pause:3.5
k8s.gcr.io/etcd:3.5.0-0
k8s.gcr.io/coredns/coredns:v1.8.4
# 根据版本构建脚本
[root@Perng-Master ~]# vi install.sh
#!/bin/bash

kubeadm config images list --kubernetes-version=v1.22.3

set -e

KUBE_VERSION=v1.22.3
KUBE_PAUSE_VERSION=3.5
ETCD_VERSION=3.5.0-0
CORE_DNS_VERSION=v1.8.4 # aliyun无该版本镜像，docker hub上有对应版本镜像

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ;
do
    docker pull $ALIYUN_URL/$imageName || docker pull coredns/coredns:1.8.4
    if [ ${imageName} != "coredns:v1.8.4" ]; # 非coredns镜像，aliyun镜像tag成k8s镜像
    then
        docker tag $ALIYUN_URL/$imageName $GCR_URL/$imageName 
    else 
        docker tag $ALIYUN_URL/$imageName $GCR_URL/coredns/$imageName
        #docker tag coredns/$imageName $ALIYUN_URL/$imageName
    fi	
    docker rmi $ALIYUN_URL/$imageName || docker rmi $imageName
done

echo
echo "docker pull finished..."

:wq
# 拉取flannel网络插件镜像 v0.16.0 
[root@Perng-Master ~]# docker pull flannelcni:v0.16.0-amd64
[root@Perng-Master ~]# docker tag flannelcni:v0.16.0-amd64 quay.io/coreos/flannel:v0.16.0
[root@Perng-Master ~]# docker rmi xwjh/flannel:v0.16.0
# 镜像打包
[root@Perng-Master ~]# docker save -o k8s.tar `docker images | grep io| awk -v  OFS=":" '{print $1,$2}'`
# 分发到各个Node节点，没有配置hosts就用ip
[root@Perng-Master ~]# scp k8s.tar Perng-Node1:/root/
[root@Perng-Master ~]# scp k8s.tar Perng-Node2:/root/
~~~

## 各Node节点载入镜像

~~~bash
[root@Perng-Node1 ~]# cd ~ 
[root@Perng-Node1 ~]# docker load -i k8s.tar 
~~~

## Master节点初始化

~~~bash
# 192.168.159.100 为master节点IP
[root@Perng-Master ~]# kubeadm init --apiserver-advertise-address=192.168.159.100 \
> --image-repository registry.aliyuncs.com/google_containers \
# 你的 K8s 版本
> --kubernetes-version=v1.22.3 \
# service的 CIDR
> --service-cidr=10.1.0.0/16 \
# Pod网络的 CIDR
> --pod-network-cidr=10.244.0.0/16

# 相关变量配置
[root@Perng-Master ~]# mkdir -p $HOME/.kube
[root@Perng-Master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@Perng-Master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 初始化后会给个命令，这个命令为Node加入集群的命令
~~~

![image-20221012171554750](/images/legacy/image-20221012171554750.png)

**如果初始化过程当中或加入集群过程中 出现访问localhost:10248超时，以下为处理办法**

![image-20221012172149524](/images/legacy/image-20221012172149524.png)

~~~bash
# 以下命令每个节点都要做
# 由于使用yum安装docker 导致cgroup错误 
# 新建 docker/daemon.json
[root@Perng-Master pki]# vim /etc/docker/daemon.json 
{"exec-opts": ["native.cgroupdriver=systemd"] }
# 重启docker
[root@Perng-Master pki]# service docker restart 
~~~



**如果出现证书错误，以下为处理方法**

![image-20221012171758521](/images/legacy/image-20221012171758521.png)

~~~bash
# 重置集群
[root@Perng-Master ~]# kubeadm reset
#删除 $HOME/./kube
[root@Perng-Master ~]# rm -rf $HOME/.kube
# 重新初始化
[root@Perng-Master ~]# kubeadm init --apiserver-advertise-address=192.168.159.100 \
> --image-repository registry.aliyuncs.com/google_containers \
> --kubernetes-version=v1.22.3 \
> --service-cidr=10.1.0.0/16 \
> --pod-network-cidr=10.244.0.0/16
#重新配置 $HOME/./kube
[root@Perng-Master ~]# mkdir -p $HOME/.kube
[root@Perng-Master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@Perng-Master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

由于集群重置，每个Node都需要重新假如集群

~~~bash
# 重置
[root@Perng-Node1 ~]# kubeadm reset
# 加入集群
[root@Perng-Node1 ~]# kubeadm join 192.168.159.100:6443 --token gbvi6q.vrgax0k4b2itg10z   --discovery-token-ca-cert-hash sha256:d267d1fa9be0ce754c6318b1afdf6b51837e0052f2e3864228057dc812f04605

~~~



## Master节点安装Flannel网络插件

~~~bash
[root@Perng-Master ~]# vim flannel-v0.16.0.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: quay.io/coreos/flannel:v0.16.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.16.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
          
          
          
:wq

[root@Perng-Master ~]# kubectl apply -f flannel-v0.16.0.yaml
[root@Perng-Master ~]# systemctl restart kubelet
~~~

## 验证

如下即可

![image-20221013140124510](/images/legacy/image-20221013140124510.png)