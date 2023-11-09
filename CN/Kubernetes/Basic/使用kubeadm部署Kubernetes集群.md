---
title: 使用kubeadm部署Kubernetes集群
date: 2019-10-24 23:25:44.0
updated: 2020-06-14 17:27:44.0
url: https://www.zhuyongci.com/archives/2019102423254471260
categories: Kubernetes
tags: Docker | Kubernetes | K8S
---

> Reference source：https://www.kubernetes.org.cn/5462.html
>
> 本文基于`kubernetes-1.14.7`版部署，容器引擎版本为`docker-ce-18.09.9`
>
> <font color="#ff0000">本文仅适用于开发和测试环境等非正式场合，不能用于生产环境！该集群的`API-Server`和`ETCD`都是**单节点**，数据有丢失风险！</font>

### 基本规划

#### 典型架构

<img src="https://oss.zhuyongci.com/bookimg/8feab8fd9dfcc566f66ea43ca5b1c54d.png!style" alt="kubernetes_three_nodes_explan" style="zoom:60%;" />

#### IP地址及主机名规划

| 主机名          | IP地址      | 备注               |
| --------------- | ----------- | ------------------ |
| `MY-K8S-MASTER` | `10.0.0.70` | `API-Server`等组件 |
| `MY-K8S-NODE1`  | `10.0.0.71` | 工作节点1          |
| `MY-K8S-NODE2`  | `10.0.0.72` | 工作节点2          |

#### 主机配置要求

| 主机            | CPU     | 内存      | 备注 |
| --------------- | ------- | --------- | ---- |
| `MY-K8S-MASTER` | 2或更高 | 2GB或更高 | 必须 |
| `MY-K8S-NODE`   | 2或更高 | 尽量大点  | 可选 |

### 准备工作

在开始部署之前，请准备好最少3台安装了Centos7的主机

#### 主机名设置

> 根据规划的主机名依次在3个节点执行

```shell
#主节点10.0.0.70
hostnamectl set-hostname MY-K8S-MASTER
#节点1
hostnamectl set-hostname MY-K8S-NODE1
#节点2
hostnamectl set-hostname MY-K8S-NODE2
```

#### 本地HOSTS解析

> 所有节点执行，如果你有自建DNS解析服务器可以忽略这步

```shell
cat <<EOF >> /etc/hosts
10.0.0.70 MY-K8S-MASTER
10.0.0.71 MY-K8S-NODE1
10.0.0.72 MY-K8S-NODE2
EOF
```

#### 宿主机系统优化

```shell
#停用防火墙
systemctl stop firewalld && systemctl disable firewalld
#关闭selinux
#立即关闭&&永久关闭
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
#关闭交换分区(公有云通常默认关闭swap，如果你的宿主机是真实的物理机则需要关闭)
#立即关闭系统所有的交换分区
swapoff -a
#永久关闭,修改/etc/fstab,将交换分区的挂载信息注释掉
sed -ri 's/^UUID(.*)swap(.*)/\#UUID\1swap\2/g' /etc/fstab
#内核参数
#内核转发及对iptables优化配置
cat > /etc/sysctl.d/k8s_network.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
#立即生效
sysctl --system
#YUM仓库配置
#首先备份本地的两个基础源
gzip /etc/yum.repos.d/CentOS-Base.repo
gzip /etc/yum.repos.d/epel.repo
#软件源优化为国内的仓库地址
#两个基础仓库
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
#配置docker仓库
curl -o /etc/yum.repos.d/docker-ce.repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#先安装epel的GPG-KEY,省得后期YUM报错NO KEY
yum install epel-release -y
#安装一些需要的工具
yum install yum-utils device-mapper-persistent-data lvm2 -y
#添加kubernetes的国内仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#清理本地的YUM缓存
yum clean all
#生成缓存(非必须)
yum makecache fast
#中间如果提示要导入KEY请选择yes
```

### 开始部署

#### 安装docker社区版

> 所有节点执行

```shell
#docker-18的最后一个版本
yum install docker-ce-18.09.9 -y
```

#### 配置docker

> 所有节点执行

配置`docker`的引擎的`cgroup`的驱动，我们使用`systemd`来对容器进程进行资源限制及管理。打开或新增`daemon.json`配置文件

```shell
#打开文件
vim /etc/docker/daemon.json
#也包含了仓库优化,加入了国内的镜像加速服务器的地址,后期如果pull镜像会先走国内
```

粘贴下面的配置，请注意`json`的语法，此处使用了[道云](https://www.daocloud.io/)提供的国内的`docker`镜像加速服务，类似的还有[阿里云](https://aliyun.com)、[网易](http://mirrors.163.com)等。

```json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": ["http://f1361db2.m.daocloud.io"]
}
```

#### 启动docker

> 所有节点执行

```shell
systemctl start docker && systemctl enable docker
```

#### 安装Kubernetes基础工具

所有节点执行以下的命令

```shell
#kubernetes-1.14的最后一个版本
yum install kubelet-1.14.7 kubeadm-1.14.7 kubectl-1.14.7 -y
#kubelet加入开机自启
systemctl enable kubelet
#注:现在不要启动kubelet,其实也根本起不来
```

#### 部署`API-Server`节点

> 在`MY-K8S-MASTER`节点执行下面的操作

1. 初始化`K8S`集群

   ```shell
   #参数解释：
   #--kubernetes-version:						版本声明
   #--apiserver-advertise-address:				指定api-server的IP地址
   #--apiserver-advertise-address:				从国内拉取镜像,必须指定,否则镜像拉不下来
   #--service-cidr:							定义VIP地址范围
   #--pod-network-cidr:						定义POD使用的网段,与flannel中的网段对应
   kubeadm init --kubernetes-version=1.14.7 --apiserver-advertise-address=10.0.0.70 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16
   ```

2. 成功后会在最后输出访问的`token`等信息，如图

   ![kubeadm_init1](https://oss.zhuyongci.com/bookimg/c3132a8fa46c930de7358d0646dd030e.png!style)

   最后的输出信息明确的告诉我们，如何管理集群以及如何把`node`节点加入`K8S`集群。

3. 记录`token`信息，后期增加节点都将使用该信息（不执行，稍后在`NODE`节点上执行）

   ```shell
   #token是唯一的,所以每个人的都不一样
   kubeadm join 10.0.0.70:6443 --token uw7o6r.sbzs7hc4hxwhnxmc \
       --discovery-token-ca-cert-hash sha256:cca7fd3b83eb3313ebc493ac1675e8678bf73ff2af503eaee9ca11295773756e 
   ```

4. 配置`kubectl`工具，用于管理集群

   ```shell
   #这里复制的kubeadm的最后输出部分
   #备注: 待添加
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   #使用kubectl检查状态
   kubectl get nodes
   #目前未配置网络插件(flannel)主节点会显示NoReady状态
   #执行完下面的第5步之后即可
   ```

5. 部署`flannel`网络插件，用于提供容器跨宿主机通信的支持

   > 项目地址：https://github.com/coreos/flannel

   ```shell
   #此处有巨坑,需要替换flannel的默认地址
   #此步骤你可以跳过,你可以直接复制我的kube-flannel.yml文件的内容
   wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   #替换为阿里的域名
   sed -i 's/quay.io\/coreos/registry.cn-beijing.aliyuncs.com\/imcto/g' kube-flannel.yml
   #注: 可以直接使用下面我修改好的配置文件
   ```

   修改`flannel`的配置，以下`kube-flannel.yml`文件内容为官方原配置修改而来，这里的配置包含了不同系统平台的相关参数，这里我们修改了所有的仓库地址，由于我们的系统平台是`arm64`平台的，所以还对网卡接口配置了一些参数。如果你的`node`节点资源包含`arm64`或其它系统平台的类型，也可以针对目标平台做针对性的配置。（注意：多网卡环境需要指定桥接的网卡的接口名称）

   ```yaml
   #File: kube-flannel.yml
   ---
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
     allowedCapabilities: ['NET_ADMIN']
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
       # SELinux is unsed in CaaSP
       rule: 'RunAsAny'
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1beta1
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
   apiVersion: rbac.authorization.k8s.io/v1beta1
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
   #修改下面的参数可定义默认使用的网段
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
     name: kube-flannel-ds-amd64
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
                     - key: beta.kubernetes.io/os
                       operator: In
                       values:
                         - linux
                     - key: beta.kubernetes.io/arch
                       operator: In
                       values:
                         - amd64
         hostNetwork: true
         tolerations:
         - operator: Exists
           effect: NoSchedule
         serviceAccountName: flannel
         initContainers:
         - name: install-cni
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-amd64
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
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-amd64
           command:
           - /opt/bin/flanneld
           args:
           - --ip-masq
           - --kube-subnet-mgr
           #如果有多块网卡,需要添加网卡接口参数,默认加上也没关系
           #有些系统未作优化,可能名称为ens33之类
           - --iface=eth0
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
                add: ["NET_ADMIN"]
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
           - name: cni
             hostPath:
               path: /etc/cni/net.d
           - name: flannel-cfg
             configMap:
               name: kube-flannel-cfg
   ---
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: kube-flannel-ds-arm64
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
                     - key: beta.kubernetes.io/os
                       operator: In
                       values:
                         - linux
                     - key: beta.kubernetes.io/arch
                       operator: In
                       values:
                         - arm64
         hostNetwork: true
         tolerations:
         - operator: Exists
           effect: NoSchedule
         serviceAccountName: flannel
         initContainers:
         - name: install-cni
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-arm64
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
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-arm64
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
                add: ["NET_ADMIN"]
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
           - name: cni
             hostPath:
               path: /etc/cni/net.d
           - name: flannel-cfg
             configMap:
               name: kube-flannel-cfg
   ---
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: kube-flannel-ds-arm
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
                     - key: beta.kubernetes.io/os
                       operator: In
                       values:
                         - linux
                     - key: beta.kubernetes.io/arch
                       operator: In
                       values:
                         - arm
         hostNetwork: true
         tolerations:
         - operator: Exists
           effect: NoSchedule
         serviceAccountName: flannel
         initContainers:
         - name: install-cni
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-arm
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
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-arm
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
                add: ["NET_ADMIN"]
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
           - name: cni
             hostPath:
               path: /etc/cni/net.d
           - name: flannel-cfg
             configMap:
               name: kube-flannel-cfg
   ---
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: kube-flannel-ds-ppc64le
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
                     - key: beta.kubernetes.io/os
                       operator: In
                       values:
                         - linux
                     - key: beta.kubernetes.io/arch
                       operator: In
                       values:
                         - ppc64le
         hostNetwork: true
         tolerations:
         - operator: Exists
           effect: NoSchedule
         serviceAccountName: flannel
         initContainers:
         - name: install-cni
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-ppc64le
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
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-ppc64le
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
                add: ["NET_ADMIN"]
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
           - name: cni
             hostPath:
               path: /etc/cni/net.d
           - name: flannel-cfg
             configMap:
               name: kube-flannel-cfg
   ---
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: kube-flannel-ds-s390x
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
                     - key: beta.kubernetes.io/os
                       operator: In
                       values:
                         - linux
                     - key: beta.kubernetes.io/arch
                       operator: In
                       values:
                         - s390x
         hostNetwork: true
         tolerations:
         - operator: Exists
           effect: NoSchedule
         serviceAccountName: flannel
         initContainers:
         - name: install-cni
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-s390x
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
           image: registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-s390x
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
                add: ["NET_ADMIN"]
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
           - name: cni
             hostPath:
               path: /etc/cni/net.d
           - name: flannel-cfg
             configMap:
               name: kube-flannel-cfg
   ```

   保存改文件后，可以直接执行(网络状态好的话)

   ```shell
   kubectl apply -f kube-flannel.yml
   ```

   >  **PS：** 建议先`pull`镜像下来，用导入或将这些镜像推到本地仓库，否则根本不知道`pull`成功没有，很多人部署失败大部分是因为拉不到镜像。
   >
   > ```shell
   > #这里针对的是网络不好的情况下的方法
   > #手动pull镜像下来,便于观察是否pull成功
   > docker pull registry.cn-beijing.aliyuncs.com/imcto/flannel:v0.11.0-amd64
   > #pull成功后再执行
   > kubectl apply -f kube-flannel.yml
   > ```

6. 查看`MY-K8S-MASTER`节点的`API-Server`是否已准备好

   ```shell
   kubectl get nodes
   ```

   如图，`master`节点已被发现

   ![master_ready](https://oss.zhuyongci.com/bookimg/ddd3d9df5a4c09fd80649d4eafddd25e.png!style)

#### 部署`Node`节点服务

> 在所有`node`节点执行下面的命令

```shell
kubeadm join 10.0.0.70:6443 --token uw7o6r.sbzs7hc4hxwhnxmc \
    --discovery-token-ca-cert-hash sha256:cca7fd3b83eb3313ebc493ac1675e8678bf73ff2af503eaee9ca11295773756e 
```

等待几分钟的时间后，登陆到`MY-K8S-MASTER`节点，执行下面的命令，以验证两个`NODE`节点是否被发现

```shell
#在master节点执行
kubectl get nodes
```

如果有下面的输出，则您的节点被加入到了集群内，此时我们的集群已经可用

![nodes_ready](https://oss.zhuyongci.com/bookimg/f2f6ce624bc45b4a9e13b77002371ec0.png!style)

> **注：** 使用`kubeadm reset`可以清除已有的集群配置信息，请谨慎使用！

### 创建测试项目

我们以`nginx`为例创建一个`K8S`的`deployment`资源，创建`nginx_dep.yaml`文件，粘贴下面的内容

```yaml
apiVersion: extensions/v1beta1
#资源类型声明
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  #副本数量
  replicas: 3
  template:
    metadata:
      labels:
      #标签定义,通过标签可关联svc资源
        app: nginx
    spec:
    #容器定义
      containers:
      - name: nginx
        image: nginx:latest
        #暴露的端口
        ports:
        - containerPort: 80
        #资源限制选项
        resources:
          limits:
            cpu: 100m
          requests:
            cpu: 100m
```

创建`Service`资源，新增`nginx_svc.yaml`文件，粘贴如下内容

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
    - port: 80
    #对外暴露的端口,通过任意节点的IP:30080即可访问容器服务
      nodePort: 30080
      targetPort: 80
  selector:
  #标签选择器,关联deployment
    app: nginx
```

将以上文件保存到独立的目录，以表明该文件是同一个项目的，具有关联性，创建资源执行下面的命令

```shell
#创建nginx的Deployment资源
kubectl create -f nginx_dep.yaml
#创建nginx的Service资源
kubectl create -f nginx_svc.yaml
```

稍等片刻，可打开浏览器访问`http://10.0.0.71:30080`即可看到`nginx`的测试页面。集群内的任意节点地址均可以访问到。

![nginx_deployment_test](https://oss.zhuyongci.com/bookimg/ba865037978545f38817e8d43f92289a.png!style)