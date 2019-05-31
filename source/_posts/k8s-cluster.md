---
title: k8s 集群搭建
date: 2019-05-27 17:18:53
categories:
- k8s
- kubernetes
- cluster
tags:
- k8s
- docker
- kubernetes
- cluster
---


## 一、主机环境预设
### 1. 测试环境说明
集群由一个master主机及两个node主机组成，分别拥有双核CPU,2G内存，系统为CentOS 7.5 1804，域名为 ilinux.io。此外各个主机预设的环境如下：
- 借助NTP服务器设定各个节点时间精确同步；
- 通过DNS完成各个节点的主机名称解析，测试环境主机数量较少时也可以使用hosts文件进行。
- 关闭各个节点的iptables或firewalld服务，并确保他们被禁止随系统引导过程启动；
- 各个节点禁用SELinux;
- 各个节点禁用所有的Swap设备；
- 若要使用ipvs模型的proxy，各个节点还要载入ipvs相关的各个模块；

### 2. 设定时钟同步
 若节点可直接访问互联网，直接启动chronyd系统服务，并设定其随系统引导而启动。
 ``` sh
yum -y install chrony

systemctl start chronyd
systemctl enable chronyd
 ```
 不过建议用户配置使用本地的时间服务器，在节点数量众多时尤其如此。存在可用的本地时间服务器时，修改节点/etc/chrony.conf配置文件，并将时间服务器执行相应的主机即可，配置格式：
 ``` sh
server CHRONY-SERVER-NAME-OR-IP iburst
 ```
 ### 3. 主机名称解析
  出于简化配置步骤的目的，本测试环境使用hosts文件进行各个节点名称解析，文件内容如下：
  ``` sh
 192.168.30.100  master1.myk8s.io  master1
192.168.30.101  node1.myk8s.io  node1
192.168.30.102  node2.myk8s.io  node2
192.168.30.103  node3.myk8s.io  node3
192.168.30.104  node4.myk8s.io  node4
192.168.30.105  node5.myk8s.io  node5
  ```
  修改完成后相应的ssh客户端需要重新登录，登录完成后可以看到终端命令显示的节点是否与实际相对应。如四号机器对应的是node4，则终端显示的是[root@node4 ~]# 。如果没有起作用则需要自行修改：
  ``` sh
  hostnamectl --static set-hostname node1
  ```
### 4. 关闭防火墙，iptables或firewalld
``` sh
  systemctl stop firewalld
  systemctl disable firewalld
```
### 5. 关闭并禁用 SELinux
若当前启用了SELinux，则需要编辑 /etc/sysconfig/selinux文件，禁用SELinux，并临时设置其当前状态为 permissive:
``` sh
sed -i 's@^\(SELINUX=\).*@\1disabled@' /etc/sysconfig/selinux
setenforce 0
```
### 6. 禁用Swap设备
``` sh
# 临时关闭
swapoff -a

# 永久关闭， 将fstab中所有swap类型的设备注释即可
vim /etc/fstab
```
### 7. 启用 ipvs内核模块
创建内核模块载入相关的脚本文件/etc/sysconfig/modules/ipvs.modules，设定自动载入的内核模块。文件内容如下：
``` sh
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
    /sbin/modinfo -F filename $i &> /dev/null
    if [ $? -eq 0 ]; then
         /sbin/modprobe $i
    fi
done
```
修改文件权限，并手动为当前系统加载内核模块：
``` sh
chmod +x /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
```

## 二、 安装程序包
### 1. 生成yum仓库配置
首先安装docker再安装k8s。通过访问网站 https://opsx.alibaba.com/mirror/ 获取docker-ce的配置仓库的配置文件：
``` sh
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
```
然后手动生成kubernetes的yum仓库配置文件 /etc/yum.repos.d/kubernetes.repo：
``` sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg/
EOF
```
### 2. 安装相关程序包
K8s 会对经过充分验证的Docker程序版本进行认证，目前认证完成的版本是 17.03，但docker-ce的最新版本已经高出了几个版本。管理员可忽略此认证而直接使用最新版本的docker-ce程序。不过建议根据说明将安装命令替换为17.03版本：
``` sh
yum install docker-ce

#安装指定版本
yum install -y --setopt=obsoletes=0 docker-ce=17.03.2.ce docker-ce-selinux-17.03.2.ce
```
开始安装 k8s ：
``` sh
yum install -y kubelet kubeadm kubectl

# 若安裝出現以下錯誤 则可以先将相关包移除后再重新执行命令
[root@node2 ~]# yum install -y kubelet kubeadm kubectl
...
Total size: 56 M
Installed size: 255 M
Downloading packages:
Running transaction check
Running transaction test

Transaction check error:
  file /usr/bin/kubectl from install of kubectl-1.14.2-0.x86_64 conflicts with file from package kubernetes-client-1.5.2-0.7.git269f928.el7.x86_64

Error Summary
-------------
[root@node2 ~]# yum remove -y kubernetes-client-1.5.2-0.7.git269f928.el7.x86_64

#设置开机启动
systemctl enable docker
systemctl enable kubelet
systemctl start docker  

```
## 三、启动各节点的docker服务，并设置开机启动
若要通过默认的 k8s.gcr.io 镜像仓库获取 Kubernetes系统组件的相关镜像，需要配置docker Unit File (/usr/lib/systemd/system/docker.service文件)中的 Environment 变量，为其定义的 HTTPS_PROXY，格式如下：
``` sh
# HTTPS_PROXY 也可以配置阿里云地址
Environment="HTTPS_PROXY=http://hub-mirror.c.163.com"
Environment="NO_PROXY=192.168.0.0/16,127.0.0.0/8"
```
也可以使用daocloud的加速地址，直接执行以下命令即可，该脚本会将加速地址放到 /etc/docker/daemon.json 文件中
``` sh
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```
另外，docker 自1.13 版起会自动设置 iptables的 FORWARD 默认策略为 DROP，这可能会影响k8s集群依赖的报文转发功能，因此需要在docker服务启动后，重新将 FORWARD 链的默认策略设备为 ACCEPT ，方式是修改 /var/lib/systemd/system/docker.service文件，在 “ExecStart=/usr/bin/dockerd”一行之后新增一行：
``` sh
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
```
重载完成后即可重启docker服务，重启完成后可以通过 docker info 查看是否设置成功：
``` sh
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart docker
[root@localhost ~]# systemctl enable docker
[root@localhost ~]# docker info
......
HTTPS Proxy: http://hub-mirror.c.163.com
No Proxy: 192.168.0.0/16,127.0.0.0/8
......

[root@localhost ~]# iptables -vnL
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
73726   39M KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
88961  127M ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   10   600 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
 8602 4830K INPUT_direct  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
 8602 4830K INPUT_ZONES_SOURCE  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
 8602 4830K INPUT_ZONES  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    2    80 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
 8590 4830K REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
......
```
有部分场景 iptables会无效，可以通过 sysctl 查看:
``` sh
[root@localhost ~]# sysctl -a |grep bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-filter-pppoe-tagged = 0
net.bridge.bridge-nf-filter-vlan-tagged = 0
net.bridge.bridge-nf-pass-vlan-input-dev = 0
.....

# 手动修改 , 在文件夹下随意创建一个文件增加两个值
[root@localhost ~]# vim /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# 使它生效
[root@localhost ~]# sysctl -p /etc/sysctl.d/k8s.conf
```

## 四、初始化主节点
开始初始化集群，初始化时系统会到google的容器仓库拉起相关镜像，国内访问较慢所以改成阿里云仓库。若之前未关闭swap设备，则需要编辑kubelet的配置文件 /etc/sysconfig/kubelet，设置其忽略swap启用的状态
``` sh
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```
设置完成首先需要单独获取相关镜像文件，再执行kubeadm init命令：
``` sh
kubeadm config images pull
```
### 初始化方式一
初始化参数直接放在命令行中执行：
``` sh
kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
```
命令各个参数说明如下：
- --kebernetes-version 指定要部署的 k8s 程序版本，需要与当前kubeadm支持的版本保持一致
- --pod-network-cidr 指定为 Pod 分配使用的网络地址，通常应该与要部署的网络插件的默认设定保持一致，如flannel， calico 等， 10.244.0.0/16 是flannel默认使用的网络
- --service-cidr 用于指定Service分配使用的网络地址，由k8s管理，默认为 10.96.0.0/12 
- --ignore-preflight-errors 忽略指定错误，如swap

### 初始化方式二
kubeadm可通过配置文件加载配置，以定制更丰富的部署选项。如以下配置文件，明确定义了kubeProxy的模式为ipvs，并支持通过修改imageRepository的值修改获取系统镜像时使用的仓库地址。该配置可以使用以下命令获得当前默认配置
``` sh
kubeadm config print init-defaults
```
``` yml
apiVersion: kubeadm.k8s.io/v1beta1
kind: MasterConfiguration
kubernetesVersion: v1.14.2
api:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
  controlPlaneEndpoint: ""
imageRepository: k8s.gcr.io
kubeProxy:
  config:
     mode: "ipvs"
     ipvs:
       ExcludeCIDRs: null
       minSyncPeriod: 0s
       scheduler: ""
       syncPeriod: 30s
kubeConfiguration:
  baseConfig:
    cgroupDriver: cgroupfs
    clusterDNS:
    - 10.96.0.10
    clusterDomian: cluster.local
    failSwapOn: false
    resolveConf: /etc/resolv.conf
    staticPodPath: /etc/kubernetes/manifests
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
```
将配置保存为 kubeadm-config.yaml，然后执行命令：
``` sh
kubeadm init --kubernetes-version=v.1.13.3 --config kubeadm-config.yaml
```
### 开始初始化主节点
下面开始执行命令：
``` sh
# 执行完成出现错误，端口被占用，通过ss或netstat命令查看哪个程序占用
[root@localhost ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.14.2
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING Swap]: running with swap on is not supported. Please disable swap
	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Port-2379]: Port 2379 is in use
	[ERROR Port-2380]: Port 2380 is in use
	[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

# 查看端口占用
[root@localhost ~]# ss -lp|grep 2379
tcp    LISTEN     0      128    127.0.0.1:2379                  *:*                     users:(("etcd",pid=1557,fd=6))
[root@localhost ~]# ss -lp|grep 2380
tcp    LISTEN     0      128    127.0.0.1:2380                  *:*                     users:(("etcd",pid=1557,fd=5))
# 关闭 etcd 应用
[root@localhost ~]# systemctl stop etcd

[root@localhost ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.14.2
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING Swap]: running with swap on is not supported. Please disable swap
	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Port-2379]: Port 2379 is in use
	[ERROR Port-2380]: Port 2380 is in use
	[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
# 出现错误 /var/lib/etcd is not empty ，直接删除后重试
[root@localhost ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap

# 执行以上命令后发现无法拉取镜像，只能通过中转方式获取相关镜像后再执行
[root@localhost ~]# kubeadm config images list
I0528 17:12:53.344118  116889 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I0528 17:12:53.344315  116889 version.go:97] falling back to the local client version: v1.14.2
k8s.gcr.io/kube-apiserver:v1.14.2
k8s.gcr.io/kube-controller-manager:v1.14.2
k8s.gcr.io/kube-scheduler:v1.14.2
k8s.gcr.io/kube-proxy:v1.14.2
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1

[root@localhost ~]# kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#docker.io/mirrorgooglecontainers#g' |sh -x
[root@localhost ~]# docker images |grep mirrorgooglecontainers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#docker.io/mirrorgooglecontainers#k8s.gcr.io#2' |sh -x
[root@localhost ~]# docker images |grep mirrorgooglecontainers |awk '{print "docker rmi ", $1":"$2}' |sh -x
# kubeadm init初始化过程中 coredns 版本可能对不上，这时就需要重新下载指定版本号的镜像。
# 我下载1.2.2版本后发现实际使用的是1.3.1，重新拉取后再执行就可以
[root@localhost ~]# docker pull coredns/coredns:1.2.2
[root@localhost ~]# docker tag coredns/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2
[root@localhost ~]# docker rmi coredns/coredns:1.2.2

# 执行后还是有问题，kubelet 没有启动
[root@localhost ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
......
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp [::1]:10248: connect: connection refused.

Unfortunately, an error has occurred:
	timed out waiting for the condition

This error is likely caused by:
	- The kubelet is not running
	- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
	- 'systemctl status kubelet'
	- 'journalctl -xeu kubelet'

Additionally, a control plane component may have crashed or exited when started by the container runtime.
To troubleshoot, list all containers using your preferred container runtimes CLI, e.g. docker.
Here is one example how you may list all Kubernetes containers running in docker:
	- 'docker ps -a | grep kube | grep -v pause'
	Once you have found the failing container, you can inspect its logs with:
	- 'docker logs CONTAINERID'
error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
# 启动 kubelet 后发现无法启动
[root@localhost ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Tue 2019-05-28 17:44:15 HKT; 9s ago
     Docs: https://kubernetes.io/docs/
  Process: 120329 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 120329 (code=exited, status=255)

# 出现以下错误的解决方式 ：  echo "1" >/proc/sys/net/bridge/bridge-nf-call-iptables
[root@master1 ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.14.2
[preflight] Running pre-flight checks
	[WARNING Swap]: running with swap on is not supported. Please disable swap
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

# 查看 kubelet 日志，发现还是需要将swap功能关闭
# 如日志没有找到相关信息，则有可能是需要调整  KUBELET_EXTRA_ARGS 参数，被这个参数坑了一天..
# 编辑文件 /etc/sysconfig/kubelet 将文件内容改为： KUBELET_EXTRA_ARGS="--fail-swap-on=false"
[root@localhost ~]# journalctl -xefu kubelet
......
May 28 21:27:21 localhost.localdomain kubelet[39754]: I0528 21:27:21.007882   39754 server.go:417] Version: v1.14.2
May 28 21:27:21 localhost.localdomain kubelet[39754]: I0528 21:27:21.008158   39754 plugins.go:103] No cloud provider specified.
May 28 21:27:21 localhost.localdomain kubelet[39754]: I0528 21:27:21.008194   39754 server.go:754] Client rotation is on, will bootstrap in background
May 28 21:27:21 localhost.localdomain kubelet[39754]: I0528 21:27:21.011019   39754 certificate_store.go:130] Loading cert/key pair from "/var/lib/kubelet/pki/kubelet-client-current.pem".
May 28 21:27:21 localhost.localdomain kubelet[39754]: I0528 21:27:21.055387   39754 server.go:625] --cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /
May 28 21:27:21 localhost.localdomain kubelet[39754]: F0528 21:27:21.055684   39754 server.go:265] failed to run Kubelet: Running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename                                Type                Size        Used        Priority /dev/dm-1                               partition        2097148        163328        -1]
May 28 21:27:21 localhost.localdomain systemd[1]: kubelet.service: main process exited, code=exited, status=255/n/a
May 28 21:27:21 localhost.localdomain systemd[1]: Unit kubelet.service entered failed state.
May 28 21:27:21 localhost.localdomain systemd[1]: kubelet.service failed.

# 关闭 swap 后启动成功（记得先做kubeadm reset）
[root@localhost ~]# swapoff -a
[root@localhost ~]# kubeadm reset
[root@localhost ~]# kubeadm init --kubernetes-version=v1.14.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
......
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.30.100:6443 --token mn7x02.m6ii9zlyo4qu3x6f \
    --discovery-token-ca-cert-hash sha256:342f43d192fcae484a9b06589af834c6b974d9d33a61fbab8cce772bcd46914f

# 接着看提示来执行，在指定用户目录下保存当前k8s集群配置文件
[root@localhost ~]# mkdir -p $HOME/.kube
[root@localhost ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@localhost ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看当前主节点状态，当前状态为 NotReady，因为还需要安装网络插件
[root@localhost ~]# kubectl get nodes
NAME                    STATUS     ROLES    AGE     VERSION
localhost.localdomain   NotReady   master   8m39s   v1.14.2

# 当 www.github.com/coreos 中找到 flannel，打开在readme中找到安装脚本执行安装
[root@localhost ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

[root@localhost ~]# kubectl get pods -n kube-system
NAME                                            READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-g8fbs                         0/1     Pending   0          16m
coredns-fb8b8dccf-w9gct                         0/1     Pending   0          16m
etcd-localhost.localdomain                      1/1     Running   0          15m
kube-apiserver-localhost.localdomain            1/1     Running   0          15m
kube-controller-manager-localhost.localdomain   1/1     Running   0          15m
kube-flannel-ds-amd64-z7qxl                     1/1     Running   0          92s
kube-proxy-b6rs9                                1/1     Running   0          16m
kube-scheduler-localhost.localdomain            1/1     Running   0          15m
[root@localhost ~]# kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
localhost.localdomain   Ready    master   17m   v1.14.2

```
## 五、初始化子节点并将其加入集群
主节点初始化完成，然后初始化子节点：
``` sh
[root@node1 ~]# systemctl enable docker.service
[root@node1 ~]# systemctl enable kubelet.service
[root@node1 ~]# systemctl start docker.service
[root@node1 ~]# systemctl start kubelet.service


# 若子节点想通过命令查看各节点状态 则需要将主节点的配置文件放置到各个子节点的Home目录的 .kube/config 文件下
root@master1 ~]# kubectl get nodes
root@master1 ~]# scp /etc/kubernetes/admin.conf root@node1:~/.kube/config

# join到集群的子节点依然需要部分镜像，如flannel ,kube-proxy ,pause .这些镜像可以在已存在镜像的主节点复制过来
# docker save $(docker image ls|egrep "k8s.gcr.io/*"|awk '{print $1}') -o k8s.master.img.tar
root@master1 ~]# docker save k8s.gcr.io/kube-proxy quay.io/coreos/flannel k8s.gcr.io/pause -o k8s.img.tar
root@master1 ~]# scp k8s.img.tar root@node1:~

# 执行加入集群命令
[root@node1 ~]# kubeadm join 192.168.30.100:6443 --token mn7x02.m6ii9zlyo4qu3x6f     --discovery-token-ca-cert-hash sha256:342f43d192fcae484a9b06589af834c6b974d9d33a61fbab8cce772bcd46914f --ignore-preflight-errors=Swap

# 查看集群节点状态
[root@master1 ~]# kubectl get nodes
NAME             STATUS     ROLES    AGE    VERSION
master1          Ready      master   17m    v1.14.2
node1.myk8s.io   Ready      <none>   2m2s   v1.14.2
node2.myk8s.io   NotReady   <none>   34s    v1.14.2
node3            Ready      <none>   81s    v1.14.2
node4            NotReady   <none>   76s    v1.14.2

# 启动完成后查看节点状态
[root@localhost ~]# kubectl describe node node4
......
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 28 May 2019 05:16:59 +0800   Tue, 28 May 2019 04:43:35 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 28 May 2019 05:16:59 +0800   Tue, 28 May 2019 04:43:35 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 28 May 2019 05:16:59 +0800   Tue, 28 May 2019 04:43:35 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Tue, 28 May 2019 05:16:59 +0800   Tue, 28 May 2019 04:43:35 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```
因为kubelet配置了network-plugin=cni，但是还没安装，所以状态会是NotReady,不想看这个报错或者不需要网络，就可以修改kubelet配置文件，去掉network-plugin=cni 就可以了。1.11.2版本之前可以修改下面的配置文件，删除最后一行里的$KUBELET_NETWORK_ARGS ：
``` sh
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
1.11.2 版本及以后的配置文件封装在/var/lib/kubelet/kubeadm-flags.env文件中
``` sh
[root@k8s ~]# cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS=--cgroup-driver=systemd --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d --network-plugin=cni
[root@localhost ~]# systemctl start kubelet
[root@localhost ~]# kubeadm reset 
```
但是生产环境还是需要的，所以我没有删除这个参数.。通过查询 node2节点的 /var/log/messages 日志，可以看到他没有拉取到镜像：
> failed: rpc error: code = Unknown desc = failed pulling image \"k8s.gcr.io/pause:3.1
所以直接把镜像从master节点复制过来即可，复制完成后发现node2已经Ready：
``` sh
[root@node2 ~]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
[root@node2 ~]# ls
anaconda-ks.cfg  k8s.img.tar  k8s.node.img.tar
[root@node2 ~]# docker load -i k8s.node.img.tar
[root@node2 ~]# kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
master1          Ready      master   34m   v1.14.2
node1.myk8s.io   Ready      <none>   18m   v1.14.2
node2.myk8s.io   Ready      <none>   17m   v1.14.2
node3            Ready      <none>   18m   v1.14.2
node4            NotReady   <none>   18m   v1.14.2
```
剩下的就是node4节点，该节点已经存在所需的镜像，错误提示：
> Unable to update cni config: No networks found in /etc/cni/net.d

该节点暂且搁置，作为实验环境三台节点已经足够。有时间再补充该错误解决方案。

