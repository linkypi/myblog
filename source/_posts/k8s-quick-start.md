---
title: Kubernetes 入门实践
date: 2019-05-14 20:59:04
categories:
- Kubernetes
tags: 
- Kubernetes
- k8s
- docker
---
&emsp;&emsp;在2016年左右就学过Docker，不过那时候感觉没什么实际用途。后来进到一家公司后才发现在使用Docker，慢慢的又恢复了对它的兴趣，也重新对它做了些了解。再后来了解到云计算、云计算的三种服务模式 IaaS（Infrastructure as a Service）, PaaS( Platform as a Service) , SaaS(Software as a Service)以及云计算平台OpenStack。

&emsp;&emsp;云计算强调的不再是实体物理机器，而是资源（如CPU、网络、存储等），通过虚拟化技术将资源整合后做细粒度管理，资源可以更加高效的被利用。我们平时在阿里云、腾讯云购买的服务器就是如此（据说谷歌云服务所使用的机器已经达到百万台），他们不需要也不会给我们准备实体机器，而是在云平台中虚拟化出咱们需要的资源：几核CPU，多少G内存，多大硬盘，带宽又是多少...如此这般我们就可以使用我们的服务器了，等我们的租期一到系统就会将这部分的资源回收，待下一个租户重新分配。 我们拿到服务器后就可以在上面部署自己的应用，但是因为公用的是同一个系统环境而无法部署多平台应用，若系统环境有问题很有可能会导致系统重启。Docker的出现使得这些问题得到很好的解决，基于Linux的Container技术实现了很好的环境隔离，容器环境有问题只需重新构建即可，无需重启机器。

&emsp;&emsp;Docker的出现带来了不少便利，但是对于当前大数据量大分布式系统盛行的时代，系统的集成、部署、资源调度、快速伸缩及版本控制等相关功能还不能很好的解决，而k8s的实现才使得应用的持续集成及部署更加便利，这里面集成了谷歌十几年来的心血。

&emsp;&emsp;技术虽好也需要看场景，杀鸡焉用牛刀，因为涉及到一系列成本。说多了手痒，很早就想学习k8s，现在有时间就来体验下。

## 安装 k8s
执行命令：

``` shell 
yum install -y etcd
yum install -y kubernetes
```

若已经安装docker新版可能会报错：

``` shell
Error: docker-ce conflicts with 2:docker-1.13.1-96.gitb2f74b2.el7.centos.x86_64
Error: docker-ce-cli conflicts with 2:docker-1.13.1-96.gitb2f74b2.el7.centos.x86_64
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```
遇到该情况可以现将已安装的版本卸载，先查看已安装版本：
``` shell
[root@localhost ~]# yum list installed|grep docker
containerd.io.x86_64                 1.2.0-3.el7                    @docker-ce-edge
docker-ce.x86_64                     3:18.09.0-3.el7                @docker-ce-edge
docker-ce-cli.x86_64                 1:18.09.0-3.el7                @docker-ce-edge
```
然后使用yum卸载即可：
``` shell
 yum remove docker-ce.x86_64
 yum remove docker-ce-cli.x86_64

 # 卸载完成后继续安装
 yum install -y kubernetes
```
## 启动服务
安装完成后按顺序启动相关服务：
``` shell
systemctl start etcd
systemctl start docker
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
```

## 创建 mysql 副本控制器 ReplicationController
``` yaml
apiVersion: v1
kind: ReplicationController                   #副本控制器RC
metadata:
  name: mysql                                         #RC的名称，全局唯一
spec:
  replicas: 1                                               #pod副本期待数量
  selector:
    app: mysql                                              #符合目标的pod拥有此标签
  template:                                                  #根据此模板创建pod的副本
    metadata:
      labels:
        app: mysql                           
    spec:
      containers:                                              #pod内容器的定义
      - name: mysql                                                   #容器的名称
        image: mysql                                                      #容器的镜像
        ports:
        - containerPort: 3306                                          #容器应用监听的端口
        env:                                                                            #注入容器内的环境变量
        - name: MYSQL_ROOT_PASSWORD                             
          value: "123456"
```
执行命令开始创建：
``` shell
kubectl create -f mysql-rc.yaml

# 查看 rc
[root@localhost test]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     1         0         0         14s

# 查询 pods
[root@localhost test]# kubectl get pods
No resources found.

```
### kubectl get pods : No resources found
出现 No resources found. 的问题可以修改apiserver文件来完成：
``` shell
vim /etc/kubernetes/apiserver
```
找到：KUBE_ADMISSION_CONTROL，去除ServiceAccount，保存退出，然后重启 kubectl
``` shell
systemctl restart kube-apiserver

[root@localhost test]# kubectl create -f mysql-rc.yaml 
replicationcontroller "mysql" created
[root@localhost test]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     1         1         0         7s
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-8qcq0   0/1       ContainerCreating   0          12s
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-8qcq0   0/1       ContainerCreating   0          19s
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-8qcq0   0/1       ContainerCreating   0          22s
```
### pod status 一直是 ContainerCreating
新的问题来了，kubectl get pods   中 status 中一直是 ContainerCreating ，这里需要在docker查询可用的pod-infrastructure并下载：

``` shell
[root@localhost test]# docker search pod-infrastructure
INDEX       NAME                                                DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/infrastructureascode/aws-cli              Containerized AWS CLI on alpine to avoid r...   10                   [OK]
docker.io   docker.io/openshift/origin-pod                      The pod infrastructure image for OpenShift 3    10                   
docker.io   docker.io/newrelic/infrastructure-k8s               NewRelic Kubernetes integration.                7                    
docker.io   docker.io/davinkevin/podcast-server                 Container around the Podcast-Server Applic...   6                    
docker.io   docker.io/newrelic/infrastructure                   Public image for New Relic Infrastructure ...   6                    
docker.io   docker.io/stefanprodan/podinfo                      Go multi-arch microservice template for Ku...   2                    
docker.io   docker.io/tianyebj/pod-infrastructure               registry.access.redhat.com/rhel7/pod-infra...   2                    
docker.io   docker.io/mosquitood/k8s-rhel7-pod-infrastructure                                                   1                    
docker.io   docker.io/podigg/podigg-lc-hobbit                   A HOBBIT dataset generator wrapper for PoDiGG   1                    [OK]
[root@localhost test]# docker pull docker.io/infrastructureascode/aws-cli
Using default tag: latest
Trying to pull repository docker.io/infrastructureascode/aws-cli ... 
latest: Pulling from docker.io/infrastructureascode/aws-cli
5a3ea8efae5d: Pull complete 
f42291af87d7: Pull complete 
b6754e713366: Pull complete 
Digest: sha256:9aab2b6ecd0637902aa7126a8be0b5fb3c84ee6b0bd60061ffe357e0b0c1735a
Status: Downloaded newer image for docker.io/infrastructureascode/aws-cli:latest

```
拉取镜像 pod-infrastructure 完成后还需要修改下文件 /etc/kubernetes/kubelet 中的镜像地址，改成当前下载的镜像地址：
``` shell
vim /etc/kubernetes/kubelet

# 找到属性 KUBELET_POD_INFRA_CONTAINER，不要在原有行修改，复制出来一行后再编辑，再将原有行注释掉。
KUBELET_POD_INFRA_CONTAINER="docker.io/infrastructureascode/aws-cli"

# 修改完成后重启服务
systemctl restart kubelet
```

接下来将之前创建的 rc 删除后重建(删除 rc 之后对应的pod会自动删除)：
``` shell
[root@localhost test]# kubectl delete rc mysql
replicationcontroller "mysql" deleted
[root@localhost test]# kubectl delete pod mysql-8qcq0
Error from server (NotFound): pods "mysql-8qcq0" not found
[root@localhost test]# kubectl get pods
No resources found.
[root@localhost test]# kubectl get rc
No resources found.
[root@localhost test]# kubectl create -f mysql-rc.yaml 
replicationcontroller "mysql" created
[root@localhost test]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     1         1         0         3s
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-mnfkj   0/1       ContainerCreating   0          11s
[root@localhost test]# kubectl logs  -f mysql-mnfkj
Error from server (BadRequest): container "mysql" in pod "mysql-mnfkj" is waiting to start: ContainerCreating
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-mnfkj   0/1       ContainerCreating   0          1m
```
pod的状态还是没有改变，看看该 pod 的日志：
``` shell
[root@localhost test]# kubectl describe pod mysql-mnfkj
Name:           mysql-mnfkj
Namespace:      default
Node:           127.0.0.1/127.0.0.1
Start Time:     Wed, 15 May 2019 06:36:37 +0800
Labels:         app=mysql
Status:         Pending
IP:             
Controllers:    ReplicationController/mysql
Containers:
  mysql:
    Container ID:       
    Image:              mysql
    Image ID:           
    Port:               3306/TCP
    State:              Waiting
      Reason:           ContainerCreating
    Ready:              False
    Restart Count:      0
    Volume Mounts:      <none>
    Environment Variables:
      MYSQL_ROOT_PASSWORD:      123456
Conditions:
  Type          Status
  Initialized   True 
  Ready         False 
  PodScheduled  True 
No volumes.
QoS Class:      BestEffort
Tolerations:    <none>
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath   Type            Reason          Message
  ---------     --------        -----   ----                    -------------   --------        ------          -------
  3m            3m              1       {default-scheduler }                    Normal          Scheduled       Successfully assigned mysql-mnfkj to 127.0.0.1
  2m            41s             3       {kubelet 127.0.0.1}                     Warning         FailedSync      Error syncing pod, skipping: failed to "StartContainer" for "POD" with ErrImagePull: "image pull failed for gcr.io/google_containers/pause-amd64:3.0, this may be because there are no credentials on this request.  details: (Get https://gcr.io/v1/_ping: dial tcp 108.177.97.82:443: connect: connection refused)"
```
通过日志可看到pod在拉取镜像的时候出了问题，那试试手动拉取：
``` shell
[root@localhost test]# docker search pause:3.0
INDEX       NAME                                        DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/monitoringartist/zabbix-3.0-xxl   Please use our better compatible Zabbix im...   77                   [OK]
docker.io   docker.io/emccorp/ecs-software-3.0.0        EMC Elastic Cloud Storage v3.0.0                5                    
docker.io   docker.io/google/pause                                                                      4                    
docker.io   docker.io/rancher/pause-amd64                                                               3                    
docker.io   docker.io/warrior/pause-amd64               pause-amd64:3.0                                 2                    [OK]
docker.io   docker.io/ibmcom/pause                      Docker Image for IBM Cloud private-CE (Com...   1                    
docker.io   docker.io/ist0ne/pause-amd64                https://gcr.io/google_containers/pause-amd64    1                    [OK]
[root@localhost test]# docker pull  docker.io/google/pause
Using default tag: latest
Trying to pull repository docker.io/google/pause ... 
latest: Pulling from docker.io/google/pause
a3ed95caeb02: Pull complete 
f72a00a23f01: Pull complete 
Digest: sha256:e8fc56926ac3d5705772f13befbaee3aa2fc6e9c52faee3d96b26612cd77556c
Status: Downloaded newer image for docker.io/google/pause:latest

[root@localhost test]# docker search pause-amd64
INDEX       NAME                                           DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/mirrorgooglecontainers/pause-amd64                                                   16                   
docker.io   docker.io/googlecontainer/pause-amd64                                                          3                    
docker.io   docker.io/rancher/pause-amd64                                                                  3                    
docker.io   docker.io/warrior/pause-amd64                  pause-amd64:3.0                                 2                    [OK]
docker.io   docker.io/cnych/pause-amd64                    gcr.io/google_containers/pause-amd64 mirro...   1                    
docker.io   docker.io/huangyj/pause-amd64                  gcr.io/google_containers/pause-amd64            1                    [OK]

[root@localhost test]# docker pull docker.io/googlecontainer/pause-amd64 
Using default tag: latest
Trying to pull repository docker.io/googlecontainer/pause-amd64 ... 
Get https://registry-1.docker.io/v2/googlecontainer/pause-amd64/manifests/latest: net/http: TLS handshake timeout
[root@localhost test]# docker pull docker.io/googlecontainer/pause-amd64 
Using default tag: latest
Trying to pull repository docker.io/googlecontainer/pause-amd64 ... 
Get https://registry-1.docker.io/v2/googlecontainer/pause-amd64/manifests/latest: net/http: TLS handshake timeout
[root@localhost test]# docker pull docker.io/warrior/pause-amd64 
Using default tag: latest
Trying to pull repository docker.io/warrior/pause-amd64 ... 
Get https://registry-1.docker.io/v2/: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
[root@localhost test]# docker pull docker.io/rancher/pause-amd64 
Using default tag: latest
Trying to pull repository docker.io/rancher/pause-amd64 ... 
manifest for docker.io/rancher/pause-amd64:latest not found

```
非常蛋疼，找不到镜像，只能找找其他供应商提供的镜像，如阿里：
``` shell
[root@localhost test]# docker pull registry.cn-shenzhen.aliyuncs.com/cp_m/pause-amd64:3.1
Trying to pull repository registry.cn-shenzhen.aliyuncs.com/cp_m/pause-amd64 ... 
3.1: Pulling from registry.cn-shenzhen.aliyuncs.com/cp_m/pause-amd64
7675586df687: Pull complete 
Digest: sha256:fcaff905397ba63fd376d0c3019f1f1cb6e7506131389edbcb3d22719f1ae54d
Status: Downloaded newer image for registry.cn-shenzhen.aliyuncs.com/cp_m/pause-amd64:3.1
[root@localhost test]# docker tag registry.cn-shenzhen.aliyuncs.com/cp_m/pause-amd64:3.1 k8s.gcr.io/pause-amd64:3.1

```
增加下docker镜像加速地址：

``` shell
[root@localhost test]# vim /etc/docker/daemon.json
# 增加了数组前面两个地址
{"registry-mirrors": ["https://registry-mirror.qiniu.com","https://zsmigm0p.mirror.aliyuncs.com","http://f1361db2.m.daocloud.io"]}

[root@localhost test]# systemctl restart docker
[root@localhost test]# systemctl enable docker   # 开机启动
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@localhost test]# systemctl enable kubelet  # 开机启动
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.


[root@localhost test]# kubectl delete rc mysql
replicationcontroller "mysql" deleted
[root@localhost test]# kubectl get pods
No resources found.
[root@localhost test]# kubectl create -f mysql-rc.yaml 
replicationcontroller "mysql" created
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-4sc9p   0/1       ContainerCreating   0          3s
[root@localhost test]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     1         1         0         12s
[root@localhost test]# kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
mysql-4sc9p   1/1       Running   0          4m
```
状态转为Running需要一些时间，请耐心等待。启动完成后来看看docker中启动的容器：
``` shell
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
a7027c839374        mysql                                      "docker-entrypoint..."   7 minutes ago       Up 7 minutes                            k8s_mysql.f6601b53_mysql-4sc9p_default_e5791240-769d-11e9-a828-000c295d1408_5a1cc99c
edd8e4b9b08e        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 7 minutes ago       Up 7 minutes                            k8s_POD.16b20365_mysql-4sc9p_default_e5791240-769d-11e9-a828-000c295d1408_bac0fd3c
```
## 创建 mysql service

创建service文件 mysql-svc.yaml：
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql   # service的全局唯一名称
spec:
  ports:
    - port: 3306
  selector:
    app: mysql   
```
``` shell
[root@localhost ~]# kubectl create -f mysql-svc.yaml 
service "mysql" created
[root@localhost ~]# kubectl get svc
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   10.254.0.1       <none>        443/TCP    1h
mysql        10.254.232.109   <none>        3306/TCP   7s

```
## 创建 web 副本控制器ReplicationController
接着创建tomcat的副本控制器文件 myweb-rc.yaml :
``` yaml
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 5
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql'
        - name: MYSQL_SERVICE_PORT
          value: '3306'
```

``` shell
[root@localhost test]# kubectl create -f myweb-rc.yaml 
replicationcontroller "myweb" created
[root@localhost test]# kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
mysql-4sc9p   1/1       Running             0          37m
myweb-0gx6n   0/1       ContainerCreating   0          10s
myweb-0pgdm   0/1       ContainerCreating   0          10s
myweb-lf0qn   0/1       ContainerCreating   0          10s
myweb-m6j0v   0/1       ContainerCreating   0          10s
myweb-zrw4b   0/1       ContainerCreating   0          10s

# 下载镜像需要几分钟
[root@localhost test]# kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
mysql-4sc9p   1/1       Running   0          40m
myweb-0gx6n   1/1       Running   0          3m
myweb-0pgdm   1/1       Running   0          3m
myweb-lf0qn   1/1       Running   0          3m
myweb-m6j0v   1/1       Running   0          3m
myweb-zrw4b   1/1       Running   0          3m
```
## 创建 web service
接着创建Service文件 myweb-svc.yaml:
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
  selector:
    app: myweb
```
执行命令: kubectl create -f myweb-srv.yaml，创建完成后看看已创建的service 及 rc

``` sh
[root@localhost ~]# kubectl get svc
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   10.254.0.1       <none>        443/TCP          17h
mysql        10.254.232.109   <none>        3306/TCP         15h
myweb        10.254.219.64    <nodes>       8080:30001/TCP   15h

[root@localhost ~]# kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
mysql     1         1         1         15h
myweb     5         5         5         15h

```
## 最后
在该虚拟机执行 curl http://192.168.175.120:30001 可以获取到网页脚本，但是我在宿主机访问这个地址怎么都无法访问到。防火墙已经开启30001端口，还是无法访问，把防火墙关闭才可以访问。进去可以看到一个tomcat页面。但是还有一个问题，就是进入demo的时候会报错 http://192.168.175.120:30001/demo/ ：
```
Error:com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
```
我把myweb和mysql容器重启后还是无效，最后进入到mysql容器看日志也没有看到错误，先这样吧。

``` sh

 [root@localhost ~]# docker restart $(docker ps -a|grep k8s_myweb.* | awk '{print $1}')
```