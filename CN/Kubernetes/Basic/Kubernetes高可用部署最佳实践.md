---
title: Kubernetes-1.17+高可用部署最佳实践
date: 2021-03-04 22:55:56.0
updated: 2021-09-01 13:12:56.0
url: https://www.zhuyongci.com/archives/kubernetes-117高可用部署最佳实践
categories: Kubernetes
tags: Kubernetes
---

###### [../  🔙](https://yongci.github.io/CN/Kubernetes/index.html)

## Kubernetes高可用部署最佳实践

[TOC]

>
> Date: 2021/01/23
>
> 仅适用于开发、测试环境，生产环境使用需要考虑网络传输性能、安全等诸多因素

### 准备工作

#### 操作系统镜像及环境说明

1. 操作系统、内核、软件包版本信息

   OS: CentOS 7-2009

   Kernel:  3.10.0

   Docker: 18.09.9

   Kubernetes: 1.17.16

   cfssl: 1.5.0

2. 操作系统安装、优化、基础组件安装，制作基础系统模板

   > 装系统步骤咱就不再提了，那个比较简单，搞不定呢就用现成的系统模板(公有云都有不少现成的模板，对于私有云来说，运维小伙伴都会做好一些优化过的虚拟机系统模板)；当然如果非要装，<font color="#ff0000">一定要关闭`swap`分区，强制要求！</font>
   >
   > 其它要求：
   >
   > 1. 网卡设备名固定为eth
   > 2. 我的环境是ESXi，版本较低，无法使用他们的NSX网络组件的集群，故纯原生版Kubernetes
   > 3. 制作的模板系统时，固定IP地址为: 172.16.5.253(修改/etc/sysconfig/ifcfg-eth0)
   > 4. 我本地已具备PowerDNS域名解析系统，如果你的环境没有，考虑部署一套
   > 5. 我的环境有现成的容器镜像仓库服务，如果你没有可以考虑暂且使用离线镜像
   > 6. 某些镜像必须科学上网，所以尽量将其拉到本地，然后再推送到本地镜像仓库

   安装基础软件，有些是依赖项有些是日常运维排障必备

   ```shell
   #修改Base源为阿里云的，加快安装速度
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
   curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
   sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
   #添加EPEL源
   curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
   #添加Docker源
   curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   yum makecache
   yum install -y yum-utils device-mapper-persistent-data lvm2 vim lrzsz net-tools socat tcpdump bind-utils wget bash-completion bash-completion-extras telnet jq yum-utils tree docker-ce.x86_64 3:18.09.9-3.el7 docker-ce-cli.x86_64 1:18.09.9-3.el7
   #上面docker-ce那个写了绝对的版本号，否则它给你装最新的版本
   ```

3. 优化Docker配置和系统配置

   ```shell
   ######永久关闭防火墙
   systemctl stop firewalld && systemctl disable firewalld
   ######关闭selinux
   #立即关闭
   setenforce 0
   #永久关闭
   sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
   ######如果没关闭交换分区，在这里再关下，kubelet要求必须关闭
   #立即关闭系统所有的交换分区
   swapoff -a
   #永久关闭,修改/etc/fstab,将交换分区的挂载信息注释掉
   sed -ri 's/^UUID(.*)swap(.*)/\#UUID\1swap\2/g' /etc/fstab
   #先启动一次，docker会在/etc里创建对应目录
   systemctl start docker
   #优化镜像源，cgroup驱动，对于cgroup驱动这个参数无论使用哪个，必须与kubelet启动参数中的保持一致，这个后面会详细说明
   #存储驱动优化及日志分页大小，防止日志文件过多耗尽系统inode
   cat >/etc/docker/daemon.json<<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2",
     "storage-opts": [
       "overlay2.override_kernel_check=true"
     ],
     "registry-mirrors": [
       "https://1b4h4c1y.mirror.aliyuncs.com",
       "https://hub-mirror.c.163.com",
       "https://reg-mirror.qiniu.com"
     ]
   }
   EOF
   #内核某些功能开启和优化
   cat >/etc/sysctl.d/kubernetes-net.conf<<EOF
   net.bridge.bridge-nf-call-iptables=1
   net.bridge.bridge-nf-call-ip6tables=1
   net.ipv4.ip_forward=1
   vm.swappiness=0
   vm.overcommit_memory=1
   vm.panic_on_oom=0
   fs.inotify.max_user_watches=89100
   EOF
   #重启Docker服务并加入开机自启、立即应用内核优化配置
   systemctl restart docker && systemctl enable docker
   sysctl --system
   ```

#### 下载要用到的所有二进制包

> 目录规划:
>
> ```shell
> #良好的习惯可以使集群更规范和便于维护
> #/opt── kubernetes
> #     ├── charts                  #存放一些集群组件的模板
> #     ├── cni                     #存放网络插件部署YAML
> #     ├── istio                   #istio部署相关YAML等
> #     ├── metrics-server          #metrics-server部署相关代码
> #     ├── packages                #集群可执行文件存放
> #     ├── psp                     #集群Pod Security Policies定义
> #     ├── README.md               #目录说明文档
> #     ├── rook                    #分布式文件系统部署相关YAML
> #     └── cluster-certs           #集群证书生成目录，仅备份
> #一键创建以上目录、文件
> mkdir -p /opt/kubernetes/{charts,cni,istio,metrics-server,packages,psp,rook,cluster-cert} && touch /opt/kubernetes/README.md
> ```

1. 下载`cfssl`系列的证书签发套件，到其官方主页的`Release`发布页下载预编译的可执行文件，仓库地址：https://github.com/cloudflare/cfssl/releases

   > 截至本文发布日起，其最新版本为`1.5.0`，下载`cfssl_1.5.0_linux_amd64`、`cfssljson_1.5.0_linux_amd64`、`certinfo_1.5.0_linux_amd64`这三个可执行文件，然后重命名、增加执行权限后移动到`/usr/local/bin`下

   ```shell
   #下载
   wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64
   wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64
   wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64
   #重命名
   mv cfssl-certinfo_1.5.0_linux_amd64 cfsslview
   mv cfssljson_1.5.0_linux_amd64 cfssljson
   mv cfssl_1.5.0_linux_amd64 cfssl
   #加执行权限
   chmod +x cfsslview cfssljson cfssl
   #移动到用户bin目录里
   mv cfsslview cfssljson cfssl /usr/local/bin/
   ```

2. 下载`Kubernetes`发行版，我们本次使用的版本为`1.17.16`，到`Release`页面定位到这个版本，然后到`CHANGELOG`中找到预编译的包，链接：https://github.com/kubernetes/kubernetes/releases/tag/v1.17.16

   ![kubernetes-changelog-1.17.16](https://oss.zhuyongci.com/bookimg/72eeeeb5d3484f7dfcd07ff6534f8f4e.png!style)

   ![kubernetes-changelog-1.17.16-pkg](https://oss.zhuyongci.com/bookimg/b8011156a90af9c8c861cf6ed9b4964d.png!style)

   执行下面的命令下载:

   ```shell
   #如果被墙了，想办法出去呗
   #从上图可以看到，Kubernetes可以在各种体系架构和OS上运行(Windows目前基于WSL2的也可以,性能较之前提升了很多)，另外一个典型的栗子：rancher整的那个k3s就主要用于边缘计算，ARM处理器架构的；在去年的kubecon2020大会上，还有人编排过Android系统
   #Server相关包(含kubeadm/kubectl/kube-apiserver/controller-manager/scheduler/kube-proxy等)
   wget -O /opt/kubernetes/packages/kubernetes-server-linux-amd64.tar.gz https://dl.k8s.io/v1.17.16/kubernetes-server-linux-amd64.tar.gz
   #Node相关包(可选)
   #wget -O /opt/kubernetes/packages/kubernetes-node-linux-amd64.tar.gz https://dl.k8s.io/v1.17.16/kubernetes-node-linux-amd64.tar.gz
   #Client for amd64(可选)
   #wget -O /opt/kubernetes/packages/kubernetes-client-linux-amd64.tar.gz https://dl.k8s.io/v1.17.16/kubernetes-client-linux-amd64.tar.gz
   #client for windows64 桌面环境用
   wget -O /opt/kubernetes/packages/kubernetes-client-windows-amd64.tar.gz https://dl.k8s.io/v1.17.16/kubernetes-client-windows-amd64.tar.gz
   #client for osx 桌面环境用
   wget -O /opt/kubernetes/packages/kubernetes-client-darwin-amd64.tar.gz https://dl.k8s.io/v1.17.16/kubernetes-client-darwin-amd64.tar.gz
   #解压
   cd /opt/kubernetes/packages
   tar xf kubernetes-server-linux-amd64.tar.gz
   #tar xf kubernetes-node-linux-amd64.tar.gz
   #tar xf kubernetes-client-linux-amd64.tar.gz
   #复制可执行文件到/usr/local/bin下
   cp kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kubeadm,kubelet,kube-proxy} /usr/local/bin/
   ```

3. 下载ETCD

   到ETCD官方仓库的Release发布页，寻找最新的稳定版，仓库地址：https://github.com/etcd-io/etcd/releases

   ```shell
   #本文使用3.4.14
   wget -O /opt/kubernetes/packages/etcd-v3.4.14-linux-amd64.tar.gz https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz
   cd /opt/kubernetes/packages/
   tar xf etcd-v3.4.14-linux-amd64.tar.gz
   cp etcd-v3.4.14-linux-amd64/{etcd,etcdctl} /usr/local/bin/
   ```

   **注：** 针对深度强迫症患者，你可以将那些特定的二进制包复制到不同节点，然后将这些节点做成不同种类的主机模板；对于`Master`节点而言需要具有：`etcd/etcdctl/kube-apiserver/kube-scheduler/kube-controller/kubectl/kubeadm`，对于`Worker`节点而言需要有：`kubelet/kube-proxy/kubectl(可选，放一个纯粹为了生成配置文件用)

4. 下载Helm(非必选，但建议在Master上装好，方便后面安装某些插件)

   后面我们需要使用Helm部署CoreDNS和Ingress-Nginx组件，这个是可选的，对于新版的Helm来说，它默认使用`kubeconfig`来操作集群，介时，你可下载windows版或OSX版的Helm，然后在你的客户机直接操作集群，此外还有Lens或不少IDE中的插件都可以直接操作集群；官方Release发布页地址：https://github.com/helm/helm/releases

   ```shell
   #请自行阅读发布描述，然后选择集群兼容的版本，这里我们使用的是helm-3.4.2
   #不同版本差异较大，请务必阅读官方说明，对于个人学习能力的提升很有帮助
   wget -O /opt/kubernetes/packages/helm-v3.4.2-linux-amd64.tar.gz https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
   cp /opt/kubernetes/packages
   tar xf helm-v3.4.2-linux-amd64.tar.gz
   cp linux-amd64/helm /usr/local/bin/
   ```

5. 下载Calico相关YAML，后面会用到

   > 官方快速上手说明：https://docs.projectcalico.org/getting-started/kubernetes/quickstart

   ```shell
   wget -O /opt/kubernetes/cni/tigera-operator.yaml https://docs.projectcalico.org/manifests/tigera-operator.yaml
   wget -O /opt/kubernetes/cni/custom-resources.yaml https://docs.projectcalico.org/manifests/custom-resources.yaml
   ```

6. 生成SSH密钥对，然后将该虚拟机保存为模板或镜像，后面需要同步几个Master节点的配置文件和证书

   ```shell
   #一路回车到底
   ssh-keygen
   #将公钥导入该模板机
   ssh-copy-id root@172.16.5.253
   ```

   现在模板机基本做的差不多了，清理下部分无用的文件，将其保存为虚拟机模板或iso镜像；然后基于该模板创建下一节的所有主机，同时修改好IP地址。

#### IP地址及主机名规划

这里将`hostname`配置为节点`ip`地址，因此无需做`hosts`文件解析或使用企业内部`DNS`服务器(后期，业务模块迁入集群是需要在本地部署`DNS`服务器的)

##### 集群节点IP

| 主机名&IP    | 部署组件                                                    | 备注                                         |
| ------------ | ----------------------------------------------------------- | -------------------------------------------- |
| 172.16.5.11  | ETCD、API-Server、Controller-Manager、Scheduler、Keepalived | 集群Master1节点、ETCD集群、Keepalived        |
| 172.16.5.12  | ETCD、API-Server、Controller-Manager、Scheduler、Keepalived | 集群Master2节点、ETCD集群                    |
| 172.16.5.13  | ETCD                                                        | ETCD集群                                     |
| 172.16.5.15  | VIP                                                         | Keepalived使用的虚拟IP，在两个Master节点上漂 |
| 172.16.5.21  | kubelet、kube-proxy                                         | Worker节点                                   |
| 172.16.5.22  | kubelet、kube-proxy                                         | Worker节点                                   |
| 172.16.5.23  | kubelet、kube-proxy                                         | Worker节点                                   |
| 172.16.5.253 | 虚拟机模板机                                                | 以上所有节点基于该模板创建                   |

**注**： 由于节点规模较小，编排的容器数量并不突出，故并未在`API-Server`前另加一层反向代理服务器；如果规模较大，`API-Server`接口请求较高，则考虑部署反向代理服务器做负载均衡；因为该环境仅用于测试，故包转发这块没有使用`ipvs`方案，还是用传统的`iptables`方案，`iptables`做包转发在`Service`数量较高的时候，内核会比较繁忙。

##### 集群服务IP

| 类型/服务      | CIDR or IP                                  |
| -------------- | ------------------------------------------- |
| Pod IP范围     | 10.244.0.0/16                               |
| Cluster IP范围 | 10.192.0.0/12                               |
| kubernetes     | 10.192.0.1(一般取ClusterIP段的第一个可用IP) |
| coredns        | 10.192.0.2(一般取ClusterIP段的第二个可用IP) |

#### 集群架构

![kubernetes-cluster-ha](https://oss.zhuyongci.com/bookimg/2c823a6aaf5c5e2ac945e09e5f420110.png!style)

### CFSSL工具集签发集群所需证书

#### 证书签发类型说明

1. Client

   此类证书通常用于客户端连接服务端时提供给服务端的凭据，`Server`侧会验证客户端的凭据但并不关心由客户端的`IP`，通常这类证书可以分发使用，没有限制；比较典型的如：`kubectl`所使用的`config`中的证书和私钥

2. Server

   此类证书主要用于客户端验证**服务端**是否来自可信IP；服务端会验证客户端是否携带有效凭据，而客户端也会验证服务端是否可信，客户端不仅要检查服务端所出示的证书有效性，还要验证对方IP是否存在于证书的`hosts`列表内，任何一个不满足要求，握手都将失败；比较典型的例子如：`API-Server`服务所使用的证书和私钥

3. Peer

   此类证书通常用于双向验证的场合，用于服务与服务之间，彼此都要包证对方绝对可信，这里会涉及对各自使用的`IP`进行验证；比如`API-Server`与`kubelet`通信时，双方都要验证对方的证书是否来自可信`IP`

   **注:** 目前在`Kubernetes`中无论通过任何方式访问集群组件的接口，都得携带凭据，任何没有凭据的访问都是耍流氓。

#### 签发集群所需所有证书

以下操作和命名均遵循Kubernetes官方推荐命名规则和存放路径，如此可保证使用某些诸如`kubeadm`等工具操作时，不会报找不到文件；在官方推荐中：`ETCD`使用了独立的`CA`证书，然后基于此根证书又签发了服务证书和给`API-Server`所使用的客户端证书；这跟国内很多人的做法不一样，国内很多人的做法是`Kubernetes`集群和`ETCD`集群使用同一套`CA`，建议用官方的方法，有助于更好的理解服务与服务之间的关系。

1. 创建证书签发类型模板和证书集中存放目录

   ```shell
   #创建证书保留目录
   #这里严格遵守了官方的命名规则，方便后期某些入kubeadm工具使用
   mkdir -p /opt/kubernetes/cluster-certs/{etcd,kubernetes-pki,front-proxy}
   mkdir -p /opt/kubernetes/cluster-certs/etcd/{kube-apiserver-etcd-client,kube-etcd,kube-etcd-healthcheck-client,kube-etcd-peer}
   mkdir -p /opt/kubernetes/cluster-certs/kubernetes-pki/{admin,controller-manager,kube-apiserver,kube-apiserver-kubelet-client,kubelet,kube-proxy,scheduler}
   mkdir -p /opt/kubernetes/cluster-certs/front-proxy/front-proxy-client
   #创建签发类型模板, 这个配置是共用的, 后面签定的所有证书几乎都使用它
   #这里可以调整有效期, 这里整的全是50年有效期的, 可以根据情况调整
   cat >/opt/kubernetes/cluster-certs/ca-config.json<<EOF
   {
       "signing": {
           "default": {
               "expiry": "438000h"
           },
           "profiles": {
               "server": {
                   "expiry": "438000h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "server auth"
                   ]
               },
               "client": {
                   "expiry": "438000h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "client auth"
                   ]
               },
               "peer": {
                   "expiry": "438000h",
                   "usages": [
                       "signing",
                       "key encipherment",
                       "server auth",
                       "client auth"
                   ]
               }
           }
       }
   }
   EOF
   ```

2. 创建ETCD相关证书

   ```shell
   #创建ETCD CA CSR签发配置, Common Name为官方默认
   cat >/opt/kubernetes/cluster-certs/etcd/ca-csr.json<<EOF
   {
       "CN": "etcd-ca",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "ca": {
         "expiry": "438000h"
     }
   }
   EOF
   #生成ETCD CA证书
   cd /opt/kubernetes/cluster-certs/etcd
   cfssl gencert -initca -config=/opt/kubernetes/cluster-certs/ca-config.json ca-csr.json | cfssljson -bare ca
   mv ca.pem ca.crt
   mv ca-key.pem ca.key
   #创建/etc/etcd目录并拷贝证书
   mkdir /etc/etcd && cp -r ca.crt ca.key /etc/etcd/
   #创建etcd的server证书CSR配置, 这个证书主要用于etcd对外提供服务时向客户端出示的证书
   #kube-apiserver会使用etcd-client证书来验证该证书是否合法且对方服务器IP是否存在于证书的hosts列表中
   #kube-apiserver使用gRPC框架与etcd通信
   cat >/opt/kubernetes/cluster-certs/etcd/kube-etcd/server-csr.json<<EOF
   {
       "CN": "kube-etcd",
       "hosts": [
           "localhost",
           "127.0.0.1",
           "172.16.5.11",
           "172.16.5.12",
           "172.16.5.13"
       ],
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai"
           }
       ]
   }
   EOF
   #生成etcd的server证书, 本质上就是根据上面的CSR-Config信息生成一个用于etcd服务的CSR和私钥, 然后拿ETCD的CA证书给这个CSR签个名, 如果你研究过非对称加密, 将会比较熟悉
   cd /opt/kubernetes/cluster-certs/etcd/kube-etcd
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/etcd/ca.crt -ca-key=/opt/kubernetes/cluster-certs/etcd/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=peer server-csr.json | cfssljson -bare server
   mv server.pem server.crt
   mv server-key.pem server.key
   cp -r server.crt server.key /etc/etcd/
   #创建ETCD集群节点间认证证书CSR配置
   cat >/opt/kubernetes/cluster-certs/etcd/kube-etcd-peer/peer-csr.json<<EOF
   {
       "CN": "kube-etcd-peer",
       "hosts": [
           "localhost",
           "127.0.0.1",
           "172.16.5.11",
           "172.16.5.12",
           "172.16.5.13"
       ],
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai"
           }
       ]
   }
   EOF
   cd /opt/kubernetes/cluster-certs/etcd/kube-etcd-peer/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/etcd/ca.crt -ca-key=/opt/kubernetes/cluster-certs/etcd/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=peer peer-csr.json | cfssljson -bare peer
   mv peer.pem peer.crt
   mv peer-key.pem peer.key
   cp -r peer.crt peer.key /etc/etcd/
   #创建ETCD存活检查相关证书, 本文暂时没有用到
   cat >/opt/kubernetes/cluster-certs/etcd/kube-etcd-healthcheck-client/healthcheck-client-csr.json<<EOF
   {
       "CN": "kube-etcd-healthcheck-client",
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   cd /opt/kubernetes/cluster-certs/etcd/kube-etcd-healthcheck-client/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/etcd/ca.crt -ca-key=/opt/kubernetes/cluster-certs/etcd/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client healthcheck-client-csr.json| cfssljson -bare healthcheck-client
   mv healthcheck-client.pem healthcheck-client.crt
   mv healthcheck-client-key.pem healthcheck-client.key
   cp -r healthcheck-client.crt healthcheck-client.key /etc/etcd/
   #创建kube-apiserver-etcd-client证书, 该证书主要用于API-Server连接ETCD时使用, 读写数据用的
   #这个证书要配到API-Server的配置中去
   cat >/opt/kubernetes/cluster-certs/etcd/kube-apiserver-etcd-client/apiserver-etcd-client-csr.json<<EOF
   {
       "CN": "kube-apiserver-etcd-client",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:masters"
           }
       ]
   }
   EOF
   cd /opt/kubernetes/cluster-certs/etcd/kube-apiserver-etcd-client/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/etcd/ca.crt -ca-key=/opt/kubernetes/cluster-certs/etcd/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client apiserver-etcd-client-csr.json| cfssljson -bare apiserver-etcd-client
   mv apiserver-etcd-client.pem apiserver-etcd-client.crt
   mv apiserver-etcd-client-key.pem apiserver-etcd-client.key
   mkdir -p /etc/kubernetes/pki && cp -r apiserver-etcd-client.crt apiserver-etcd-client.key /etc/kubernetes/pki/
   #至此, ETCD的相关证书都生成完毕
   ```

3. 生成Kubernetes集群相关证书

   ```shell
   #创建集群 CA CSR签发配置, Common Name为官方默认
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/ca-csr.json<<EOF
   {
       "CN": "kubernetes-ca",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "ca": {
         "expiry": "438000h"
     }
   }
   EOF
   #签发Kubernetes集群CA证书, 该证书为集群组件与组件之间相互认证使用
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/
   cfssl gencert -initca -config=/opt/kubernetes/cluster-certs/ca-config.json ca-csr.json | cfssljson -bare ca
   mv ca.pem ca.crt
   mv ca-key.pem ca.key
   cp -r ca.crt ca.key /etc/kubernetes/pki/
   #创建kube-apiserver签发配置，此证书用于kube-apiserver对外出示，所有的客户端会验证apiserver的证书与其自身绑定的IP是否匹配
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kube-apiserver/apiserver-csr.json<<EOF
   {
       "CN": "kube-apiserver",
       "hosts": [
           "127.0.0.1",
           "172.16.5.11",
           "172.16.5.12",
           "172.16.5.13",
           "172.16.5.15",
           "10.192.0.1",
           "kubernetes",
           "kubernetes.default",
           "kubernetes.default.svc",
           "kubernetes.default.svc.cluster",
           "kubernetes.default.svc.cluster.local"
       ],
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   #签发apiserver证书
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/kube-apiserver
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=server apiserver-csr.json| cfssljson -bare apiserver
   mv apiserver.pem apiserver.crt
   mv apiserver-key.pem apiserver.key
   cp -r apiserver.crt apiserver.key /etc/kubernetes/pki/
   #签发kube-apiserver-kubelet-client证书, 该证书在kube-apiserver连接kubelet时向对方出示
   #创建apiserver-kubelet-client签发配置
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kube-apiserver-kubelet-client/apiserver-kubelet-client-csr.json<<EOF
   {
       "CN": "kube-apiserver-kubelet-client",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:masters"
           }
       ]
   }
   EOF
   #签发apiserver-kubelet-client证书
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/kube-apiserver-kubelet-client/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client kube-apiserver-kubelet-client-csr.json| cfssljson -bare apiserver-kubelet-client
   mv apiserver-kubelet-client.pem apiserver-kubelet-client.crt
   mv apiserver-kubelet-client-key.pem apiserver-kubelet-client.key
   cp -r apiserver-kubelet-client.crt apiserver-kubelet-client.key /etc/kubernetes/pki/
   #创建controller-manager证书签发配置
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/controller-manager/controller-manager-csr.json<<EOF
   {
       "CN": "system:kube-controller-manager",
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   #签发controller-manager证书
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/controller-manager/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client controller-manager-csr.json| cfssljson -bare controller-manager
   mv controller-manager.pem controller-manager.crt
   mv controller-manager-key.pem controller-manager.key
   cp -r controller-manager.crt controller-manager.key /etc/kubernetes/pki/
   #创建scheduler签发配置
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/scheduler/scheduler.json<<EOF
   {
       "CN": "system:kube-scheduler",
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   #开始签发证书
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/scheduler/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client scheduler-csr.json| cfssljson -bare scheduler
   mv scheduler.pem scheduler.crt
   mv scheduler-key.pem scheduler.key
   cp -r scheduler.crt scheduler.key /etc/kubernetes/pki/
   #创建kube-proxy配置, 这个后面所有Node节点用, kube-proxy
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kube-proxy/kube-proxy-csr.json<<EOF
   {
       "CN": "system:kube-proxy",
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/kube-proxy/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client kube-proxy-csr.json| cfssljson -bare kube-proxy
   mv kube-proxy.pem kube-proxy.crt
   mv kube-proxy-key.pem kube-proxy.key
   cp -r kube-proxy.crt kube-proxy.crt /etc/kubernetes/pki/
   #此步是可选的, kube-proxy启动时会从它自己的kubeconfig文件加载证书
   scp -r kube-proxy.crt kube-proxy.key 172.16.5.21:/etc/kubernetes/pki/
   scp -r kube-proxy.crt kube-proxy.key 172.16.5.22:/etc/kubernetes/pki/
   scp -r kube-proxy.crt kube-proxy.key 172.16.5.23:/etc/kubernetes/pki/
   #创建集群管理员证书, 主要用于kubectl
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/admin/admin-csr.json<<EOF
   {
       "CN": "kubernetes-admin",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:masters"
           }
       ]
   }
   EOF
   #生成证书
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client admin-csr.json| cfssljson -bare admin
   mv admin.pem admin.crt
   mv admin-key.pem admin.key
   cp -r admin.crt admin.key /etc/kubernetes/pki/
   #创建3个node节点的kubelet证书
   #172.16.5.21
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kubelet/kubelet-csr-172.16.5.21.json<<EOF
   {
       "CN": "system:node:172.16.5.21",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:nodes"
           }
       ]
   }
   EOF
   #172.16.5.22
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kubelet/kubelet-csr-172.16.5.22.json<<EOF
   {
       "CN": "system:node:172.16.5.22",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:nodes"
           }
       ]
   }
   EOF
   #172.16.5.23
   cat >/opt/kubernetes/cluster-certs/kubernetes-pki/kubelet/kubelet-csr-172.16.5.23.json<<EOF
   {
       "CN": "system:node:172.16.5.23",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "names": [
           {
               "C": "CN",
               "ST": "ShangHai",
               "L": "ShangHai",
               "O": "system:nodes"
           }
       ]
   }
   EOF
   #生成所有节点证书
   cd /opt/kubernetes/cluster-certs/kubernetes-pki/kubelet/
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client kubelet-csr-172.16.5.21.json| cfssljson -bare kubelet-172.16.5.21
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client kubelet-csr-172.16.5.22.json| cfssljson -bare kubelet-172.16.5.22
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.crt -ca-key=/opt/kubernetes/cluster-certs/kubernetes-pki/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client kubelet-csr-172.16.5.23.json| cfssljson -bare kubelet-172.16.5.23
   #分发到所有node上
   scp -r kubelet-172.16.5.21.pem 172.16.5.21:/etc/kubernetes/pki/kubelet.crt
   scp -r kubelet-172.16.5.21-key.pem 172.16.5.21:/etc/kubernetes/pki/kubelet.key
   scp -r kubelet-172.16.5.22.pem 172.16.5.22:/etc/kubernetes/pki/kubelet.crt
   scp -r kubelet-172.16.5.22-key.pem 172.16.5.22:/etc/kubernetes/pki/kubelet.key
   scp -r kubelet-172.16.5.23.pem 172.16.5.23:/etc/kubernetes/pki/kubelet.crt
   scp -r kubelet-172.16.5.23-key.pem 172.16.5.23:/etc/kubernetes/pki/kubelet.key
   ```

4. 生成Front-Proxy相关证书

   ```shell
   #front-proxy
   cat >/opt/kubernetes/cluster-certs/front-proxy/ca-csr.json<<EOF
   {
       "CN": "kubernetes-front-proxy-ca",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "ca": {
         "expiry": "438000h"
     }
   }
   EOF
   #生成CA证书
   cd /opt/kubernetes/cluster-certs/front-proxy/
   cfssl gencert -initca -config=/opt/kubernetes/cluster-certs/ca-config.json ca-csr.json | cfssljson -bare ca
   mv ca.pem ca.crt
   mv ca-key.pem ca.key
   cp -r ca.crt ca.key /etc/kubernetes/pki/
   #创建front-proxy-client
   cat >/opt/kubernetes/cluster-certs/front-proxy/front-proxy-client/front-proxy-client-csr.json<<EOF
   {
       "CN": "front-proxy-client",
       "key": {
           "algo": "rsa",
           "size": 2048
       }
   }
   EOF
   #生成证书
   cfssl gencert -ca=/opt/kubernetes/cluster-certs/front-proxy/front-proxy-client/ca.crt -ca-key=/opt/kubernetes/cluster-certs/front-proxy/front-proxy-client/ca.key -config=/opt/kubernetes/cluster-certs/ca-config.json -profile=client front-proxy-client-csr.json| cfssljson -bare front-proxy-client
   mv front-proxy-client.pem front-proxy-client.crt
   mv front-proxy-client-key.pem front-proxy-client.key
   cp -r front-proxy-client.crt front-proxy-client.key /etc/kubernetes/pki/
   ```

5. 同步证书到各个Master节点

   ```shell
   scp -r /etc/etcd 172.16.5.12:/etc/
   scp -r /etc/kubernetes 172.16.5.12:/etc/
   scp -r /etc/etcd 172.16.5.12:/etc/
   scp -r /etc/kubernetes 172.16.5.13:/etc/
   ```



### 生成服务配置和启动服务

#### 配置&启动ETCD服务&开机自启

1. 创建`systemd`启动配置，在3台Master节点上执行如下内容

   ```shell
   #NODE_NAME替换为节点IP
   cat >/usr/lib/systemd/system/etcd.service<<EOF
   [Unit]
   Description=Etcd Server
   After=network.target
   After=network-online.target
   Wants=network-online.target
   Documentation=https://github.com/coreos
   
   [Service]
   Type=notify
   WorkingDirectory=/var/lib/etcd/
   ExecStart=/usr/local/bin/etcd \
     --data-dir=/var/lib/etcd \
     --name=${NODE_NAME} \
     --initial-advertise-peer-urls=https://${NODE_NAME}:2380 \
     --listen-peer-urls=https://${NODE_NAME}:2380 \
     --listen-client-urls=https://${NODE_NAME}:2379,https://127.0.0.1:2379 \
     --advertise-client-urls=https://${NODE_NAME}:2379 \
     --initial-cluster-token=etcd-cluster-0 \
     --initial-cluster=172.16.5.11=https://172.16.5.11:2380,172.16.5.12=https://172.16.5.12:2380,172.16.5.13=https://172.16.5.13:2380 \
     --initial-cluster-state=new \
     --client-cert-auth \
     --trusted-ca-file=/etc/etcd/ca.crt \
     --cert-file=/etc/etcd/server.crt \
     --key-file=/etc/etcd/server.key \
     --peer-client-cert-auth \
     --peer-trusted-ca-file=/etc/etcd/ca.crt \
     --peer-cert-file=/etc/etcd/peer.crt \
     --peer-key-file=/etc/etcd/peer.key
     
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 重新加载`systemd`的配置，启动服务并加入开机自启

   ```shell
   systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service
   ```

3. 观察服务是否启动

   ```shell
   systemctl status etcd.service
   ```

   ![etcd-status](https://oss.zhuyongci.com/bookimg/f20a0a482c4fbff31b94894e9759fad0.png!style)

#### 配置&启动Master节点相关服务&开机自启

##### API-Server

1. 创建`systemd`启动配置，在3台Master执行

   ```shell
   #NODE_NAME替换为对应节点的IP
   cat >/usr/lib/systemd/system/kube-apiserver.service<<EOF
   [Unit]
   Description=Kubernetes API Server
   Documentation=https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
   After=network.target etcd.target
   
   [Service]
   ExecStart=/usr/local/bin/kube-apiserver \
     --advertise-address=${NODE_NAME} \
     --bind-address=0.0.0.0 \
     --allow-privileged=true \
     --anonymous-auth=false \
     --audit-log-maxage=30 \
     --audit-log-maxbackup=3 \
     --audit-log-maxsize=100 \
     --audit-log-path=/var/log/kubernetes/kube-apiserver-audit.log \
     --authorization-mode=Node,RBAC \
     --client-ca-file=/etc/kubernetes/pki/ca.crt \
     --admission-control=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,PodSecurityPolicy,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
     --enable-aggregator-routing=true \
     --enable-bootstrap-token-auth=true \
     --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \
     --etcd-cafile=/etc/etcd/ca.crt \
     --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
     --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
     --etcd-servers=https://172.16.5.11:2379,https://172.16.5.12:2379,https://172.16.5.13:2379 \
     --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
     --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
     --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt \
     --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key \
     --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
     --kubelet-https=true \
     --kubelet-timeout=10s \
     --kubelet-preferred-address-types=InternalIP,InternalDNS \
     --logtostderr=true \
     --requestheader-extra-headers-prefix=X-Remote-Extra- \
     --requestheader-group-headers=X-Remote-Group \
     --requestheader-username-headers=X-Remote-User \
     --runtime-config=api/all=true \
     --secure-port=6443 \
     --service-account-key-file=/etc/kubernetes/pki/ca.crt \
     --service-cluster-ip-range=10.192.0.0/12 \
     --service-node-port-range=30000-32767 \
     --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
     --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
   
   Restart=on-failure
   RestartSec=5
   Type=notify
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 重新载入`systemd`配置，然后启动服务并加入开机自启

   ```shell
   systemctl daemon-reload && systemctl enable kube-apiserver.service && systemctl start kube-apiserver.service
   ```

3. 观察各节点服务状态

   ```shell
   systemctl status kube-apiserver.service
   ```

   ![kube-apiserver-status](https://oss.zhuyongci.com/bookimg/022facce0047c10dcd54a608fe52c331.png!style)

##### Controller-Manager

1. 创建`systemd`启动配置文件，在3台主机上执行

   ```shell
   cat >/usr/lib/systemd/system/kube-controller-manager.service<<EOF
   [Unit]
   Description=Kubernetes Controller Manager
   Documentation=https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/
   
   [Service]
   ExecStart=/usr/local/bin/kube-controller-manager \
     --bind-address=127.0.0.1 \
     --cluster-name=kubernetes \
     --client-ca-file=/etc/kubernetes/pki/ca.crt \
     --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
     --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
     --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
     --controllers=*,bootstrapsigner,tokencleaner \
     --experimental-cluster-signing-duration=87600h \
     --kubeconfig=/etc/kubernetes/controller-manager.conf \
     --leader-elect=true \
     --logtostderr=true \
     --root-ca-file=/etc/kubernetes/pki/ca.crt \
     --tls-cert-file=/etc/kubernetes/pki/controller-manager.crt \
     --tls-private-key-file=/etc/kubernetes/pki/controller-manager.key \
     --secure-port=10257 \
     --service-account-private-key-file=/etc/kubernetes/pki/ca.key \
     --service-cluster-ip-range=10.192.0.0/12 \
     --use-service-account-credentials=true
   
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 创建`controller-manager`的`kubeconfig`启动配置

   ```shell
   #controller-manager
   KUBECONFIG=/etc/kubernetes/controller-manager.conf
   kubectl config set-cluster default-cluster --server=https://127.0.0.1:6443 --certificate-authority /etc/kubernetes/pki/ca.crt --embed-certs --kubeconfig $KUBECONFIG
   kubectl config set-credentials default-controller-manager --client-key /etc/kubernetes/pki/controller-manager.key --client-certificate /etc/kubernetes/pki/controller-manager.crt --embed-certs --kubeconfig $KUBECONFIG
   kubectl config set-context default-system --cluster default-cluster --user default-controller-manager --kubeconfig $KUBECONFIG
   kubectl config use-context default-system --kubeconfig $KUBECONFIG
   ```

3. 重新载入`systemd`配置，启动服务并加入开机自启

   ```shell
   systemctl daemon-reload && systemctl enable kube-controller-manager.service && systemctl start kube-controller-manager.service
   ```

4. 观察服务状态

   ```shell
   systemctl status kube-controller-manager
   ```

   ![kube-controller-manager-status](https://oss.zhuyongci.com/bookimg/5f6a47944a10f7365dd02ade06676bff.png!style)

##### Scheduler

1. 创建`systemd`启动配置文件，在3台主机执行

   ```shell
   cat >/usr/lib/systemd/system/kube-scheduler.service<<EOF
   [Unit]
   Description=Kubernetes Scheduler
   Documentation=https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/
   
   [Service]
   ExecStart=/usr/local/bin/kube-scheduler \
     --bind-address=127.0.0.1 \
     --kubeconfig=/etc/kubernetes/scheduler.conf \
     --leader-elect=true \
     --logtostderr=true \
     --address=127.0.0.1 \
     --port=10251 \
     --secure-port=10259
   
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 创建`kube-scheduler`的`kubeconfig`配置文件，在3台Master执行

   ```shell
   #目标节点必须具有kubectl
   KUBECONFIG=/etc/kubernetes/scheduler.conf
   kubectl config set-cluster default-cluster --server=https://127.0.0.1:6443 --certificate-authority /etc/kubernetes/pki/ca.crt --embed-certs --kubeconfig $KUBECONFIG
   kubectl config set-credentials default-scheduler --client-key /etc/kubernetes/pki/scheduler.key --client-certificate /etc/kubernetes/pki/scheduler.crt --embed-certs --kubeconfig $KUBECONFIG
   kubectl config set-context default-system --cluster default-cluster --user default-scheduler --kubeconfig $KUBECONFIG
   kubectl config use-context default-system --kubeconfig $KUBECONFIG
   ```

3. 重新载入`systemd`配置，启动服务并加入开机自启

   ```shell
   systemctl daemon-reload && systemctl enable kube-scheduler.service && systemctl start kube-scheduler.service
   ```

4. 查看服务状态

   ```shell
   systemctl status kube-scheduler.service
   ```

   ![kube-scheduler-status](https://oss.zhuyongci.com/bookimg/eafb8d60bda8db2d1aed156b294dcb92.png!style)

##### keepalived

> `keepalived`服务主要负责`VIP`漂移，通过调用一个特定的脚本，根据脚本的退出状态来判断服务健康状况；在工作过程中会一直发送`VRRP`报文，其可以工作在**抢占式**和**非抢占式**模式，主要有以下区别:
>
> **抢占式**
>
> 在该模式下，`Master`一旦挂掉，`Backup`会立即接管；当`Master`服务恢复之后，此时会发生`VIP`漂移；此工作模式，可能会造成一定的闪断
>
> **非抢占式**
>
> 在该模式下，两台机器的身份都是`Backup`，权重比设定略有不同，发生`VIP`漂移后，此时问题节点如果恢复了，也不会立即发生新的漂移

1. 创建检测脚本，在当前主节点

   ```shell
   cat /etc/keepalived/check-apiserver-healthz.sh<<EOF
   #!/bin/sh
   
   errExit() {
      echo "*** $*" 1>&2
      exit 1
   }
   echo "off"
   curl --silent --max-time 2 http://127.0.0.1:8080/healthz -o /dev/null || errExit "Error GET https://localhost:6443/"
   if ip addr | grep -q 172.16.5.15; then
      curl --silent --max-time 2 --insecure https://172.16.5.15:6443/healthz -o /dev/null || errExit "Error GET https://172.16.5.15:6443/"
   fi
   EOF
   ```

2. 添加`keepalived`配置文件

   ```shell
   #我们使用的是抢占模式，这个在初始Master执行
   cat >/etc/keepalived/keepalived.conf<<EOF
   global_defs {
       router_id lb-apiserver
   }
   
   vrrp_script check-apiserver-healthz {
   	script "/etc/keepalived/check-apiserver-healthz.sh"
   	interval 5
   	;Run once in 5 seconds
   }
   
   vrrp_instance VI-API-SERVER {
       state MASTER
       priority 150
       dont_track_primary
       interface eth0
       virtual_router_id 94
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 0128
       }
       virtual_ipaddress {
           172.16.5.15
       }
       track_script {
           check-apiserver-healthz
       }
   }
   EOF
   #在Backup主机执行
   cat >/etc/keepalived/keepalived.conf<<EOF
   global_defs {
       router_id lb-apiserver
   }
   
   vrrp_script check-apiserver-healthz {
           ;探测脚本位置
           script "/etc/keepalived/check-apiserver-healthz.sh"
           interval 5
           ;Run once in 5 seconds
   }
   
   vrrp_instance VI-API-SERVER {
       ;角色
       state BACKUP
       ;权重比
       priority 90
       dont_track_primary
       interface eth0
       virtual_router_id 94
       advert_int 1
       authentication {
           auth_type PASS
           auth_pass 0128
       }
       virtual_ipaddress {
           172.16.5.15
       }
       track_script {
           check-apiserver-healthz
       }
   }
   EOF
   ```

3. 启动服务并加入开机自启

   ```shell
   systemctl start keepalived.service && systemctl enable keepalived.service
   ```

#### 配置&启动Worker节点相关服务&开机自启

##### kube-proxy

1. 创建开机自启动配置

   ```shell
   #NODE_NAME为节点的IP地址
   cat >/usr/lib/systemd/system/kube-proxy.service<<EOF
   [Unit]
   Description=Kubernetes Kube-Proxy Server
   Documentation=https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
   After=network.target
   
   [Service]
   WorkingDirectory=/var/lib/kube-proxy
   ExecStart=/usr/local/bin/kube-proxy \
     --kubeconfig=/etc/kubernetes/kube-proxy.conf \
     --config=/etc/kubernetes/kube-proxy-current.conf \
     --logtostderr=true \
     --hostname-override=${NODE_NAME} \
     --proxy-mode=iptables \
     --cluster-cidr=10.192.0.0/12 \
     --proxy-port-range=30000-32767
   
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 创建额外配置文件

   ```shell
   #NODE_NAME为节点IP地址
   cat >/etc/kubernetes/kube-proxy-current.conf<<EOF
   apiVersion: kubeproxy.config.k8s.io/v1alpha1
   bindAddress: ${NODE_NAME}
   clientConnection:
     acceptContentTypes: ""
     burst: 10
     contentType: application/vnd.kubernetes.protobuf
     kubeconfig: /etc/kubernetes/kube-proxy.conf
     qps: 5
   clusterCIDR: 10.192.0.0/12
   configSyncPeriod: 15m0s
   conntrack:
     maxPerCore: 32768
     min: 131072
     tcpCloseWaitTimeout: 1h0m0s
     tcpEstablishedTimeout: 24h0m0s
   enableProfiling: false
   healthzBindAddress: ${NODE_NAME}:10256
   hostnameOverride: ${NODE_NAME}
   kind: KubeProxyConfiguration
   metricsBindAddress: 127.0.0.1:10249
   mode: iptables
   nodePortAddresses: null
   oomScoreAdj: -999
   portRange: 30000-32767
   udpIdleTimeout: 250ms
   winkernel:
     enableDSR: false
     networkName: ""
     sourceVip: ""
   EOF
   ```

3. 创建`kubeconfig`配置，`kube-proxy`主要通过这个里面的凭据连接`API-SERVER`

   ```shell
   #KUBERNETES_APISERVER: master集群VIP
   #KUBERNETES_APISERVER_PORT: master集群安全端口
   #KUBERNETES_CERT_DIR: 集群证书存放目录
   #KUBERNETES_CONF_DIR: 集群配置文件存放目录
   echo -e "\033[33m生成kube-proxy配置文件: 1. 设置集群CA证书\033[0m\n"
   kubectl config set-cluster default-cluster --server=https://${KUBERNETES_APISERVER_VIP}:${KUBERNETES_APISERVER_PORT} --certificate-authority ${KUBERNETES_CERT_DIR}/ca.crt --embed-certs --kubeconfig ${KUBERNETES_CONF_DIR}/kube-proxy.conf
   echo -e "\033[33m生成kube-proxy配置文件: 2. 设置认证证书和私钥\033[0m\n"
   kubectl config set-credentials kube-proxy --client-key ${KUBERNETES_CERT_DIR}/kube-proxy.key --client-certificate ${KUBERNETES_CERT_DIR}/kube-proxy.crt --embed-certs --kubeconfig  ${KUBERNETES_CONF_DIR}/kube-proxy.conf
   echo -e "\033[33m生成kube-proxy配置文件: 3. 关联上下文\033[0m\n"
   kubectl config set-context default-system --cluster default-cluster --user kube-proxy --kubeconfig ${KUBERNETES_CONF_DIR}/kube-proxy.conf
   echo -e "\033[33m生成kube-proxy配置文件: 4. 设置默认Context\033[0m\n"
   kubectl config use-context default-system --kubeconfig $KUBECONFIG ${KUBERNETES_CONF_DIR}/kube-proxy.conf
   ```

4. 启动服务并加入开机自启

   ```shell
   systemctl start kube-proxy.service && systemctl enable kube-proxy.service
   ```

5. 观察服务状态

   ```shell
   systemctl status kube-proxy.service
   ```

   ![kube-proxy-status](https://oss.zhuyongci.com/bookimg/34f68f12f5b50c628651dd2fd1c227e3.png!style)

##### kubelet

1. 添加`systemd`的启动配置

   ```shell
   #NODE_NAME替换为节点IP地址
   cat >/usr/lib/systemd/system/kubelet.service<<EOF
   [Unit]
   Description=Kubernetes Kubelet
   Documentation=https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
   After=docker.service
   Requires=docker.service
   
   [Service]
   WorkingDirectory=/var/lib/kubelet
   ExecStart=/usr/local/bin/kubelet \
     --cgroup-driver=systemd \
     --cert-dir=/etc/kubernetes/pki \
     --client-ca-file=/etc/kubernetes/pki/ca.crt \
     --kubeconfig=/etc/kubernetes/kubelet.conf \
     --logtostderr=true \
     --network-plugin=cni \
     --hostname-override=${NODE_NAME} \
     --address=${NODE_NAME} \
     --cluster-dns=10.192.0.2 \
     --cluster-domain=cluster.local \
     --pod-infra-container-image=pause-amd64:3.1
   
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. 创建`kubelet`配置文件

   ```shell
   #节点上必须已经签发了针对于该节点的kubelet证书
   #KUBERNETES_APISERVER: master集群VIP
   #KUBERNETES_APISERVER_PORT: master集群安全端口
   #KUBERNETES_CERT_DIR: 集群证书存放目录
   #KUBERNETES_CONF_DIR: 集群配置文件存放目录
   KUBECONFIG=/etc/kubernetes/kubelet.conf
   echo -e "\033[34m生成kubelet配置文件: 1. 设置集群CA证书\033[0m\n"
   kubectl config set-cluster default-cluster --server=https://${KUBERNETES_APISERVER}:${KUBERNETES_APISERVER_PORT} --certificate-authority ${KUBERNETES_CERT_DIR}/ca.crt --embed-certs --kubeconfig ${KUBERNETES_CONF_DIR}/kubelet.conf
   echo -e "\033[34m生成kubelet配置文件: 2. 设置认证证书和私钥\033[0m\n"
   kubectl config set-credentials kubelet --client-key ${KUBERNETES_CERT_DIR}/kubelet.key --client-certificate ${KUBERNETES_CERT_DIR}/kubelet.crt --embed-certs --kubeconfig  ${KUBERNETES_CONF_DIR}/kubelet.conf
   echo -e "\033[34m生成kubelet配置文件: 3. 关联上下文\033[0m\n"
   kubectl config set-context default-system --cluster default-cluster --user kubelet --kubeconfig ${KUBERNETES_CONF_DIR}/kubelet.conf
   echo -e "\033[34m生成kubelet配置文件: 4. 设置默认Context\033[0m\n"
   kubectl config use-context default-system --kubeconfig $KUBECONFIG ${KUBERNETES_CONF_DIR}/kubelet.conf
   ```

3. 重新载入`systemd`配置，启动服务并加入开机自启

   ```shell
   systemctl daemon-reload
   systemctl enable kubelet.service && systemctl start kubelet.service
   ```

4. 观察服务状态

   ```shell
   #图不重要，看字
   systemctl statsu kubelet.service
   ```

   ![kubelet-status](https://oss.zhuyongci.com/bookimg/b6830ee015b6f91a4e1782c92a5eb942.png!style)

#### 生成kubectl配置

没啥特别的顺序，这一步在前边执行也可以的

```shell
#kubectl
KUBECONFIG=/etc/kubernetes/admin.conf
kubectl config set-cluster default-cluster --server=https://172.16.5.15:6443 --certificate-authority /etc/kubernetes/pki/ca.crt --embed-certs --kubeconfig $KUBECONFIG
kubectl config set-credentials kube-admin --client-key /etc/kubernetes/pki/admin.key --client-certificate /etc/kubernetes/pki/admin.crt --embed-certs --kubeconfig $KUBECONFIG
kubectl config set-context default-system --cluster default-cluster --user kube-admin --kubeconfig $KUBECONFIG
kubectl config use-context default-system --kubeconfig $KUBECONFIG
\cp /etc/kubernetes/admin.conf ~/.kube/config
```

#### 使用helm部署集群必要服务

##### 安装CoreDNS组件

对于Helm3变化还是蛮大，具体变化可以到https://helm.sh查看。

1. 默认情况下，coredns的镜像从国外拉，比较慢，最好翻墙；建议先把`charts`拉下来，修改下`values.yaml`

   ```shell
   #先添加官方repo
   helm repo add coredns https://coredns.github.io/helm
   helm repo update
   #拉charts下来
   helm pull coredns/coredns
   #解压这个包, 修改values.yaml, 优化其中的镜像为本地仓库的最好
   ```

2. 安装，在可以执行`kubectl`命令的机器上执行

   ```shell
   helm install kube-dns coredns/coredns -f values.yaml -n kube-system
   ```

##### 安装Ingress-Nginx组件

1. 添加仓库

   ```shell
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   ```

2. 安装

   ```shell
   helm install ingress-nginx ingress-nginx/ingress-nginx -n kube-system
   ```

##### 安装kube-dashboard(待更新)

##### 使用Rook一键部署Ceph集群(待更新)

##### 安装Prometheus-Operator(待更新)





### 附录

#### 节点自动初始化并加入集群的脚本

> 该用了很多重复的命令，建议写成函数，然后多个地方复用；期待起到抛砖引玉的效果
>
> 文章太长，已改用附件形式放该脚本

https://oss.zhuyongci.com/bookimg/81944ebbc59b0c12edd787413b387383.sh
