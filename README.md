# 一、需求介绍
安装K3S集群，并通过Kuboard对集群进行统一管理，对常用的操作进行示例展示。

# 二、安装 Docker
### 2.1 系统要求
Docker 支持 64 位版本 CentOS 7/8，并且要求内核版本不低于 3.10。 CentOS 7 满足最低内核的要求，但由于内核版本比较低，部分功能（如 overlay2 存储层驱动）无法使用，并且部分功能可能不太稳定。

### 2.2 卸载旧版本

旧版本的 Docker 称为 docker 或者 docker-engine，使用以下命令卸载旧版本：

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

### 2.3 使用 yum 安装
执行以下命令安装依赖包：

```bash
sudo yum install -y yum-utils
```

鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。

执行下面的命令添加 yum 软件源：

```bash
sudo yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

```bash
sudo sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo

# 官方源
sudo yum-config-manager \
#     --add-repo \
#     https://download.docker.com/linux/centos/docker-ce.repo
```

### 2.4 安装 Docker
更新 yum 软件源缓存，并安装 docker-ce。

```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

### 2.5 使用脚本自动安装
在测试或开发环境中 Docker 官方为了简化安装流程，提供了一套便捷的安装脚本，CentOS 系统上可以使用这套脚本安装，另外可以通过 `--mirror` 选项使用国内源进行安装：
> 若你想安装测试版的 Docker, 请从 test.docker.com 获取脚本

```bash
# curl -fsSL test.docker.com -o get-docker.sh
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
# sudo sh get-docker.sh --mirror AzureChinaCloud
```

执行这个命令后，脚本就会自动的将一切准备工作做好，并且把 Docker 的稳定(stable)版本安装在系统中。
### 2.6 启动 Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 2.7 建立 docker 用户组
默认情况下，docker 命令会使用 Unix socket 与 Docker 引擎通讯。而只有 root 用户和 docker 组的用户才可以访问 Docker 引擎的 Unix socket。出于安全考虑，一般 Linux 系统上不会直接使用 root 用户。因此，更好地做法是将需要使用 docker 的用户加入 docker 用户组。
建立 docker 组：

```bash
sudo groupadd docker
```

将当前用户加入 docker 组：

```bash
sudo usermod -aG docker $USER
```

退出当前终端并重新登录，进行如下测试。
### 2.8 测试 Docker 是否安装正确

```bash
docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
d1725b59e92d: Pull complete
Digest: sha256:0add3ace90ecb4adbf7777e9aacf18357296e799f81cabc9fde470971e499788
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

若能正常输出以上信息，则说明安装成功。
### 2.9 镜像加速
如果在使用过程中发现拉取 Docker 镜像十分缓慢，可以配置 Docker 国内镜像加速。
> Ubuntu 16.04+、Debian 8+、CentOS 7+

目前主流 Linux 发行版均已使用 `systemd` 进行服务管理，这里介绍如何在使用 `systemd` 的 Linux 发行版中配置镜像加速器。

请首先执行以下命令，查看是否在 `docker.service` 文件中配置过镜像地址。

```bash
systemctl cat docker | grep '\-\-registry\-mirror'
```

如果该命令有输出，那么请执行 `systemctl cat docker` 查看 `ExecStart=` 出现的位置，修改对应的文件内容去掉 `--registry-mirror` 参数及其值，并按接下来的步骤进行配置。

如果以上命令没有任何输出，那么就可以在 `/etc/docker/daemon.json` 中写入如下内容（如果文件不存在请新建该文件）：

```bash
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

注意，一定要保证该文件符合 json 规范，否则 Docker 将不能启动。

之后重新启动服务。

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
参照：[https://yeasy.gitbook.io/docker_practice/install/mirror](https://yeasy.gitbook.io/docker_practice/install/mirror)

# 三、安装K3s 集群安装
### 3.1 介绍
#### K3s - 轻量级 Kubernetes
轻量级 Kubernetes。安装简单，内存只有一半，所有的二进制都不到 100MB。

适用于：

- 边缘计算-Edge
- 物联网-IoT
- CI
- Development
- ARM
- 嵌入 K8s
- 不想深陷 k8s 运维管理的人
#### 什么是 K3s?
K3s 是一个完全符合 Kubernetes 的发行版，有以下增强功能。

- 打包为单个二进制文件。
- 基于 sqlite3 的轻量级存储后端作为默认存储机制。 etcd3，MySQL，Postgres 仍然可用。
- 封装在简单的启动程序中，该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。
- 添加了简单但功能强大的“batteries-included”功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik ingress controller。
- 所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中。这使 K3s 可以自动化和管理复杂的集群操作，例如分发证书。
- 外部依赖性已最小化（仅需要现代内核和 cgroup 挂载）。 K3s 软件包需要依赖项，包括：
	-  	containerd
	-  	Flannel
	-  	CoreDNS
	-  	CNI
	-  	主机实用程序 (iptables, socat, etc)
	-  	Ingress controller (traefik)
	-  	嵌入式 service loadbalancer
	-  	嵌入式 network policy controller

官网：[https://docs.rancher.cn/docs/k3s/_index/](https://docs.rancher.cn/docs/k3s/_index/)
### 3.2 server（默认嵌入式DB）
K3s 提供了一个安装脚本，可以方便的在 systemd 或 openrc 的系统上将其作为服务安装。这个脚本可以在 https://get.k3s.io 获得。要使用这种方法安装 K3s，只需运行：

```bash
curl -sfL https://get.k3s.io | sh -
```

> 国内用户，可以使用以下方法加速安装：

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

**`我使用的安装命令，通过INSTALL_K3S_EXEC="--kube-apiserver-arg=feature-gates=RemoveSelfLink=false添加启动了nfs-client-provisioner`**

> --kube-apiserver-arg=feature-gates=RemoveSelfLink=false 开启nfs存储，k8s 1.20后如果要使用nfs需要开启

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_EXEC="--kube-apiserver-arg=feature-gates=RemoveSelfLink=false"  INSTALL_K3S_MIRROR=cn sh -
```

获取`K3S_TOKEN`

`K3S_TOKEN`使用的值存储在你的服务器节点上的`/var/lib/rancher/k3s/server/node-token`

```bash
cat /var/lib/rancher/k3s/server/node-token
```

###  3.3 woker
要在工作节点上安装并将它们添加到集群，请使用K3S_URL和K3S_TOKEN环境变量运行安装脚本。这是显示如何加入工作者节点的示例：

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.3.105:6443 K3S_TOKEN=K10a22ef8983794f525ad6de879878166f3b4aeb6990874c5a5062310a4a3159cd9::server:85d16190f8a4edb9d02ea92005ca4ec2 sh -
```
设置`K3S_URL`参数会使 `K3s` 以 `worker` 模式运行。`K3s agent` 将在所提供的 URL 上向监听的 `K3s` 服务器注册。

# 四、安装 Kuboard v3.0
### 4.1 介绍
Kubernetes 容器编排已越来越被大家关注，然而使用 Kubernetes 的门槛却依然很高，主要体现在这几个方面：

- 集群的安装复杂，出错概率大

- Kubernetes相较于容器化，引入了许多新的概念，学习难度高

- 需要手工编写 YAML 文件，难以在多环境下管理

- 缺少好的实战案例可以参考

Kuboard，是一款免费的 Kubernetes 图形化管理工具，Kuboard 力图帮助用户快速在 Kubernetes 上落地微服务。

Kuboard v3.0 支持 Kubernetes 多集群管理

官网：[https://kuboard.cn/overview/](https://kuboard.cn/overview/)

### 4.2 安装 Kuboard v3 - 内建用户库
安装 Kuboard v3.0 版本的指令如下：

```bash
sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 10080:80/tcp \
  -p 10081:10081/udp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://内网IP:10080" \
  -e KUBOARD_AGENT_SERVER_UDP_PORT="10081" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
  # 也可以使用镜像 swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard:v3 ，可以更快地完成镜像下载。
  # 请不要使用 127.0.0.1 或者 localhost 作为内网 IP \
  # Kuboard 不需要和 K8S 在同一个网段，Kuboard Agent 甚至可以通过代理访问 Kuboard Server \
```
    
>KUBOARD_ENDPOINT 参数的作用是，让部署到 Kubernetes 中的 kuboard-agent 知道如何访问 Kuboard Server；
KUBOARD_ENDPOINT 中也可以使用外网 IP；
Kuboard 不需要和 K8S 在同一个网段，Kuboard Agent 甚至可以通过代理访问 Kuboard Server；
建议在 KUBOARD_ENDPOINT 中使用域名；
如果使用域名，必须能够通过 DNS 正确解析到该域名，如果直接在宿主机配置 /etc/hosts 文件，将不能正常运行；
参数解释


>建议将此命令保存为一个 shell 脚本，例如 start-kuboard.sh，后续升级 Kuboard 或恢复 Kuboard 时，需要通过此命令了解到最初安装 Kuboard 时所使用的参数；
第 4 行，将 Kuboard Web 端口 80 映射到宿主机的 10080 端口（您可以根据自己的情况选择宿主机的其他端口）；
第 5、6 行，将 Kuboard Agent Server 的端口 10081/udp、10081/tcp 映射到宿主机的 10081 端口（您可以根据自己的情况选择宿主机的其他端口）；
第 7 行，指定 KUBOARD_ENDPOINT 为 http://内网IP，如果后续修改此参数，需要将已导入的 Kubernetes 集群从 Kuboard 中删除，再重新导入；
第 8、9 行，指定 KUBOARD_AGENT_SERVER 的端口为 10081，此参数与第 5、6 行中的宿主机端口应保持一致，修改此参数不会改变容器内监听的端口 10081；
第 10 行，将持久化数据 /data 目录映射到宿主机的 /root/kuboard-data 路径，请根据您自己的情况调整宿主机路径；

### 4.3 访问 Kuboard v3.0
在浏览器输入 http://your-host-ip:10080 即可访问 Kuboard v3.0 的界面，登录方式：

>用户名： admin
密 码： Kuboard123

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223115545461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
### 4.4 添加集群
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223115637290.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223115706415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223115803288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223120042154.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223120057729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
### 4.5 在k3s集群启用其自带ingress—traefik的web-ui
#### 开启traefik的web-ui
>默认情况下k3s安装traefik没有启用其dashboard，traefik-dashboard-ingress.yaml

见[github上traefik的示例](https://github.com/traefik/traefik/blob/v1.7/examples/k8s/traefik-deployment.yaml)：

```bash
kind: Deployment
# 略
spec:
  # 略
  template:
    # 略
    spec:
      # 略
      containers:
      - image: traefik:v1.7
        # 略
        args:
        - --api
        # 略
```
即，增加traefik启动参数--api
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225104117382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
#### 配置ingress
traefik-dashboard-ingress.yaml

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
spec:
  rules:
  - host: traefik.dracula.io
    http:
      paths:
      - backend:
          serviceName: traefik
          servicePort: dash
        path: /ui
        pathType: ImplementationSpecific
```

```bash
kubectl apply -f traefik-dashboard-ingress.yaml
```
访问：[http://traefik.dracula.io/ui/dashboard/](http://traefik.dracula.io/ui/dashboard/)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225111655339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)


# 五、安装NFS
### 5.1 介绍
Kubernetes 对 Pod 进行调度时，以当时集群中各节点的可用资源作为主要依据，自动选择某一个可用的节点，并将 Pod 分配到该节点上。在这种情况下，Pod 中容器数据的持久化如果存储在所在节点的磁盘上，就会产生不可预知的问题，例如，当 Pod 出现故障，Kubernetes 重新调度之后，Pod 所在的新节点上，并不存在上一次 Pod 运行时所在节点上的数据。

为了使 Pod 在任何节点上都能够使。用同一份持久化存储数据，我们需要使用网络存储的解决方案为 Pod 提供 数据卷。常用的网络存储方案有：NFS/cephfs/glusterfs

### 5.2 配置NFS服务器
>本节中所有命令都以 root 身份执行

执行以下命令安装 nfs 服务器所需的软件包

```bash
yum install -y rpcbind nfs-utils
```
执行命令 `vim /etc/exports`，创建 exports 文件，文件内容如下：

```bash
/root/nfs_root/ *(insecure,rw,sync,no_root_squash)
```
 执行以下命令，启动 nfs 服务

```bash
# 创建共享目录，如果要使用自己的目录，请替换本文档中所有的 /root/nfs_root/
mkdir /root/nfs_root

systemctl enable rpcbind
systemctl enable nfs-server

systemctl start rpcbind
systemctl start nfs-server
exportfs -r
```
检查配置是否生效

```bash
exportfs
```

```bash
# 输出结果如下所示
/root/nfs_root /root/nfs_root
```

### 5.3 在客户端测试nfs
> 服务器端防火墙开放111、662、875、892、2049的 tcp / udp 允许，否则远端客户无法连接。

执行以下命令安装 nfs 客户端所需的软件包

```bash
yum install -y nfs-utils
```
    
执行以下命令检查 nfs 服务器端是否有设置共享目录

```bash
# showmount -e $(nfs服务器的IP)
showmount -e 192.168.3.128
# 输出结果如下所示
Export list for 192.168.3.128
/root/nfs_root *
```

执行以下命令挂载 nfs 服务器上的共享目录到本机路径 /root/nfsmount

```bash
mkdir /root/nfsmount
# mount -t nfs $(nfs服务器的IP):/root/nfs_root /root/nfsmount
mount -t nfs 172.17.216.82:/root/nfs_root /root/nfsmount
# 写入一个测试文件
echo "hello nfs server" > /root/nfsmount/test.txt
```
    
在 nfs 服务器上执行以下命令，验证文件写入成功

```bash
cat /root/nfs_root/test.txt
```

# 六、在K3S创建存储卷声明
[https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/)

### 6.1  持久化存储卷和声明介绍
PersistentVolume（PV）用于为用户和管理员提供如何提供和消费存储的API，PV由管理员在集群中提供的存储。它就像Node一样是集群中的一种资源。PersistentVolume 也是和存储卷一样的一种插件，但其有着自己独立的生命周期。PersistentVolumeClaim (PVC)是用户对存储的请求，类似于Pod消费Node资源，PVC消费PV资源。Pod能够请求特定的资源(CPU和内存)，声明请求特定的存储大小和访问模式。PV是一个系统的资源，因此没有所属的命名空间。

### 6.2 持久化存储卷和声明的生命周期
在Kubernetes集群中，PV 作为存储资源存在。PVC 是对PV资源的请求和使用，也是对PV存储资源的”提取证”，而Pod通过PVC来使用PV。PV 和 PVC 之间的交互过程有着自己的生命周期，这个生命周期分为5个阶段：

- 供应(Provisioning)：即PV的创建，可以直接创建PV（静态方式），也可以使用StorageClass动态创建
- 绑定（Binding）：将PV分配给PVC
- 使用（Using）：Pod通过PVC使用该Volume
- 释放（Releasing）：Pod释放Volume并删除PVC
- 回收（Reclaiming）：回收PV，可以保留PV以便下次使用，也可以直接从云存储中删除

根据上述的5个阶段，存储卷的存在下面的4种状态：

- Available：可用状态，处于此状态表明PV以及准备就绪了，可以被PVC使用了。
- Bound：绑定状态，表明PV已被分配给了PVC。
- Released：释放状态，表明PVC解绑PV，但还未执行回收策略。
- Failed：错误状态，表明PV发生错误。
#### 6.2.1 供应（Provisioning）
供应是为集群提供可用的存储卷，在Kubernetes中有两种持久化存储卷的提供方式：静态或者动态。

#### 6.2.1.1 静态(Static)
PV是由Kubernetes的集群管理员创建的，PV代表真实的存储，PV提供的这些存储对于集群中所有的用户都是可用的。它们存在于Kubernetes API中，并可被Pod作为真实存储使用。在静态供应的情况下，由集群管理员预先创建PV，开发者创建PVC和Pod，Pod通过PVC使用PV提供的存储。静态供应方式的过程如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225160436902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)


#### 6.2.1.2 动态（Dynamic）
对于动态的提供方式，当管理员创建的静态PV都不能够匹配用户的PVC时，集群会尝试自动为PVC提供一个存储卷，这种提供方式基于StorageClass。在动态提供方向，PVC需要请求一个存储类，但此存储类必须有管理员预先创建和配置。集群管理员需要在API Server中启用DefaultStorageClass的接入控制器。动态供应过程如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225160453560.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)


 

### 6.2.2 绑定
在Kubernetes中，会动态的将PVC与可用的PV的进行绑定。在kubernetes的Master中有一个控制回路，它将监控新的PVC，并为其查找匹配的PV（如果有），并把PVC和PV绑定在一起。如果一个PV曾经动态供给到了一个新的PVC，那么这个回路会一直绑定这个PV和PVC。另外，用户总是能得到它们所要求的存储，但是volume可能超过它们的请求。一旦绑定了，PVC绑定就是专属的，无论它们的绑定模式是什么。

如果没有匹配的PV，那么PVC会无限期的处于未绑定状态，一旦存在匹配的PV，PVC绑定此PV。比如，就算集群中存在很多的50G的PV，需要100G容量的PVC也不会匹配满足需求的PV。直到集群中有100G的PV时，PVC才会被绑定。PVC基于下面的条件绑定PV，如果下面的条件同时存在，则选择符合所有要求的PV进行绑定:

1）如果PVC指定了存储类，则只会绑定指定了同样存储类的PV；

2）如果PVC设置了选择器，则选择器去匹配符合的PV;

3）如果没有指定存储类和设置选取器，PVC会根据存储空间容量大小和访问模式匹配符合的PV。

### 6.2.3 使用
Pod把PVC作为卷来使用，Kubernetes集群会通过PVC查找绑定的PV，并将其挂接至Pod。对于支持多种访问方式的卷，用户在使用 PVC 作为卷时，可以指定需要的访问方式。一旦用户拥有了一个已经绑定的PVC，被绑定的PV就归该用户所有。用户能够通过在Pod的存储卷中包含的PVC，从而访问所占有的PV。

### 6.2.４释放
当用户完成对卷的使用时，就可以利用API删除PVC对象了，而且还可以重新申请。删除PVC后，对应的持久化存储卷被视为“被释放”，但这时还不能给其他的PVC使用。之前的PVC数据还保存在卷中，要根据策略来进行后续处理。

### 6.2.5 回收
PV的回收策略向集群阐述了在PVC释放卷时，应如何进行后续工作。目前可以采用三种策略：保留，回收或者删除。保留策略允许重新申请这一资源。在PVC能够支持的情况下，删除策略会同时删除卷以及AWS EBS/GCE PD或者Cinder卷中的存储内容。如果插件能够支持，回收策略会执行基础的擦除操作（rm -rf /thevolume/*），这一卷就能被重新申请了。

### 6.2.5.1 保留
保留回收策略允许手工回收资源。当PVC被删除，PV将仍然存储，存储卷被认为处于已释放的状态。但是，它对于其他的PVC是不可用的，因为以前的数据仍然保留在数据中。管理员能够通过下面的步骤手工回收存储卷：

1）删除PV：在PV被删除后，在外部设施中相关的存储资产仍然还在；

2）手工删除遗留在外部存储中的数据；

3）手工删除存储资产，如果需要重用这些存储资产，则需要创建新的PV。

### 6.2.5.2 循环
警告：此策略将会被遗弃。建议后续使用动态供应的模式。

循环回收会在存储卷上执行基本擦除命令：rm -rf /thevolume/*，使数据对于新的ＰＶＣ可用。

### 6.2.5.3 删除
对于支持删除回收策略的存储卷插件，删除即会从Kubernetes中移除PV，也会从相关的外部设施中删除存储资产，例如AWS EBS, GCE PD, Azure Disk或者Cinder存储卷。
## 6.3 K3S自带 Local Storage Provider
>**`K3s 自带 Rancher 的 Local Path Provisioner`**

>**`会引起NFS静态挂载失败`**

这使得能够使用各自节点上的本地存储来开箱即用地创建持久卷声明。下面我们介绍一个简单的例子。有关更多信息，请参考此处的官方文档。

创建一个由 `hostPath` 支持的持久卷声明和一个使用它的 `pod`：

pvc.yaml

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

pod.yaml

```bash
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: local-path-pvc
```

应用 yaml:

```bash
kubectl create -f pvc.yaml
kubectl create -f pod.yaml
```

确认 PV 和 PVC 已创建：

```bash
kubectl get pv
kubectl get pvc
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225115948251.png)

状态应该都为 `Bound`

## 6.4 解决K3s 自带 Rancher 的 Local Path Provisioner造成的静态挂载失败问题
> 需对`local-path-provisioner`进行卸载

具体卸载及安装方法请参见：[https://github.com/rancher/local-path-provisioner/blob/master/README.md#Uninstall](https://github.com/rancher/local-path-provisioner/blob/master/README.md#Uninstall)

## 6.5 启用动态供应
### 6.5.1 创建PV
配置文件 nfs-pv1.yml

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: "/root/nfs_root"
    server: 192.168.3.128
    readOnly: false
```
① capacity 指定 PV 的容量为 1G。

② accessModes 指定访问模式为 ReadWriteOnce，支持的访问模式有：
ReadWriteOnce – PV 能以 read-write 模式 mount 到单个节点。
ReadOnlyMany – PV 能以 read-only 模式 mount 到多个节点。
ReadWriteMany – PV 能以 read-write 模式 mount 到多个节点。

③ persistentVolumeReclaimPolicy 指定当 PV 的回收策略为 Recycle，支持的策略有：
Retain – 需要管理员手工回收。
Recycle – 清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*。
Delete – 删除 Storage Provider 上的对应存储资源，例如 AWS EBS、GCE PD、Azure Disk、OpenStack Cinder Volume 等。

④ storageClassName 指定 PV 的 class 为 nfs。相当于为 PV 设置了一个分类，PVC 可以指定 class 申请相应 class 的 PV。

⑤ 指定 PV 在 NFS 服务器上对应的目录

### 6.5.2 创建 PVC 
配置文件 nfs-pvc1.yml

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
```
PVC 就很简单了，只需要指定 PV 的容量，访问模式和 class。

从 kubectl get pvc 和 kubectl get pv 的输出可以看到 mypvc1 已经 Bound 到 mypv1，申请成功。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225164207975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
参照：[https://www.cnblogs.com/benjamin77/p/9944268.html](https://www.cnblogs.com/benjamin77/p/9944268.html)


## 6.6 部署 NFS Provisioner 为 NFS 提供动态分配卷
环境说明：
NFS Provisioner Github 地址：[https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
NFS 示例部署文件 Github 地址：[https://github.com/my-dlq/blog-example/tree/master/nfs-provisioner-deploy](https://github.com/my-dlq/blog-example/tree/master/nfs-provisioner-deploy)
## 6.6.1 NFS Provisioner 简介
NFS Provisioner 是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷。

持久卷被配置为：`namespace-{namespace}-namespace−${pvcName}-${pvName}`。
## 6.6.2 创建 NFS Server 端
本篇幅是具体介绍如何部署 NFS 动态卷分配应用 “NFS Provisioner”，所以部署前请确认已经存在 NFS Server 端，关于如何部署 NFS Server 请看之前写过的博文 “CentOS7 搭建 NFS 服务器”，如果非 Centos 系统，请先自行查找 NFS Server 安装方法。

这里 NFS Server 环境为：

IP地址：192.168.3.128
存储目录：/root/nfs_root

## 6.6.3 创建 ServiceAccount
现在的 Kubernetes 集群大部分是基于 RBAC 的权限控制，所以创建一个一定权限的 ServiceAccount 与后面要创建的 “NFS Provisioner” 绑定，赋予一定的权限。

### 6.6.6.1 清理rbac授权

如果曾经配置过nfs的授权，先清理再创建。第一次创建则不需要执行清理步骤

```bash
kubectl delete -f nfs-rbac.yaml -n kube-system
```
### 6.6.6.2 创建授权
提前修改里面的 Namespace 值设置为要部署 “NFS Provisioner” 的 Namespace 名

nfs-rbac.yaml

```bash
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

创建 RBAC

-n: 指定应用部署的 Namespace

```bash
kubectl apply -f nfs-rbac.yaml -n kube-system
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225165722408.png)

```bash
kubectl describe sa nfs-client-provisioner -n kube-system
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210226090049533.png)


### 6.6.4 部署 NFS Provisioner
设置 NFS Provisioner 部署文件，这里将其部署到 “kube-system” Namespace 中。

nfs-provisioner-deploy.yaml

```bash
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate      #---设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-client     #--- nfs-provisioner的名称，以后设置的storageclass要和这个保持一致
            - name: NFS_SERVER
              value: 192.168.3.128  #---NFS服务器地址，和 valumes 保持一致
            - name: NFS_PATH
              value: /root/nfs_root #---NFS服务器目录，和 valumes 保持一致
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.3.128   #---NFS服务器地址
            path: /root/nfs_root    #---NFS服务器目录
```
清理NFS Provisioner
>如果之前配置过可用下面的命令清理

```bash
kubectl delete -f nfs-provisioner-deploy.yaml -n kube-system
```

-n: 指定应用部署的 Namespace

```bash
kubectl apply -f nfs-provisioner-deploy.yaml -n kube-system
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225165957114.png)
```bash
kubectl describe deployment nfs-client-provisioner -n kube-system
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210226083526548.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

```bash
#查看状态
kubectl get pod -o wide -n kube-system|grep nfs-client
#查看pod日志
kubectl logs -f `kubectl get pod -o wide -n kube-system|grep nfs-client|awk '{print $1}'` -n kube-system
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210226090828280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

### 6.6.5 创建 NFS SotageClass
>创建StorageClass主要是绑定我们创建的NFS Provisioner ，在配置的是否需要注意provisioner 参数的名字要和上面我们配置nfs-provisioner-deploy.yaml文件中定义的PROVISIONER_NAME名字相同。
>
创建一个 StoageClass，声明 NFS 动态卷提供者名称为 “nfs-storage”。

nfs-storage.yaml

```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" #---设置为默认的storageclass
provisioner: nfs-client    #---动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
  archiveOnDelete: "true"  #---设置为"false"时删除PVC不会保留数据,"true"则保留数据
```
如果之前配置可用下面命令清除

```bash
kubectl delete -f nfs-storage.yaml
```

创建 StorageClass

```bash
kubectl apply -f nfs-storage.yaml
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021022517020155.png)

```bash
kubectl describe sc nfs-storage -n kube-system
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021022608391521.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

### 6.6.6 创建 PVC 和 Pod 进行测试
#### 6.6.6.1 创建测试 PVC
在 “kube-public” Namespace 下创建一个测试用的 PVC 并观察是否自动创建是 PV 与其绑定。

test-pvc.yaml

```bash
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  storageClassName: nfs-storage #---需要与上面创建的storageclass的名称一致
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
```

创建 PVC

-n：指定创建 PVC 的 Namespace

```bash
kubectl apply -f test-pvc.yaml -n kube-public
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225170335184.png)

#### 6.6.6.3 查看 PVC 状态是否与 PV 绑定

利用 Kubectl 命令获取 pvc 资源，查看 STATUS 状态是否为 “Bound”。

```bash
kubectl get pvc test-pvc -n kube-public
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210226093611610.png)
#### 6.6.6.4 创建测试 Pod 并绑定 PVC
创建一个测试用的 Pod，指定存储为上面创建的 PVC，然后创建一个文件在挂载的 PVC 目录中，然后进入 NFS 服务器下查看该文件是否存入其中。

test-pod.yaml

```bash
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:latest
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"  #创建一个名称为"SUCCESS"的文件
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
```

创建 Pod

-n：指定创建 Pod 的 Namespace

```bash
kubectl apply -f test-pod.yaml -n kube-public
```

3、进入 NFS Server 服务器验证是否创建对应文件
进入 NFS Server 服务器的 NFS 挂载目录，查看是否存在 Pod 中创建的文件：

```bash
cd /root/nfs_root
ls -l
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210226093831226.png)
参照：[https://blog.csdn.net/qq_32641153/article/details/94730358](https://blog.csdn.net/qq_32641153/article/details/94730358)
# 七、kubectl 常用命令
列出所有容器

```bash
kubectl get po
```

```bash
[root@master ~]# kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
springboot-b7554bcb8-sbj7x          1/1     Running   0          2d2h
volume-test                         1/1     Running   0          2d1h
nginx-deployment-77bd8c5d65-jnk5h   1/1     Running   0          3h27m
```
查看容器日志
```bash
kubectl logs -f volume-test
```
```bash
[root@master ~]# kubectl logs -f volume-test
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```

# 八、外部treafik使用
>待补充
