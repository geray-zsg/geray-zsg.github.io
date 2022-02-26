## 1、前置知识点
### 1.1 生成环境下可部署k8s的两种方式

- kubeadm

Kubeadm是一个K8s部署工具，提供kubeadm init和kubeadm join，用于快速部署Kubernetes集群。

- 二进制包

从github下载发行版的二进制包，手动部署每个组件，组成Kubernetes集群。
**kubeadm工具功能**

- `kubeadm init`：初始化一个master节点
- `kubeadm join`：将工作节点加入集群
- `kubeadm upgrade`：升级k8s版本
- `kubeadm token：`管理`kubeadm join`使用的令牌
- `kubeadm reset`：清空`kubeadm init`和`kubeadm join`对主机所做的任何更改
- `kubeadm version`：打印`kubeadm`版本
- `kubeadm alpha`：预览可用的新功能
### 1.2 准备环境
**服务器配置：**

- 建议最小配置：2核CPU、2G内存、20G硬盘
- 最好可以连接外网，方便拉取镜像，不能，提前下载镜像导入节点

> 我本机的配置稍高一点，所以我这里也配置的高一点 
>
> **注意：**处理器内核总数不能大于本机的逻辑处理器数

**软件环境：**

| 软件 | 版本 |
| --- | --- |
| OS | CentOS7.9_x64 （mini） |
| Docker | 20-ce（以最新为准） |
| Kubernetes | 1.21 |

**服务器规划：**

| 角色 | IP |
| --- | --- |
| k8s-master1 | 192.168.6.20 |
| k8s-node1 | 192.168.6.21 |
| k8s-node2 | 192.168.6.22 |


### 1.3 操作系统初始化配置

静态IP

```
# 配置文件/etc/sysconfig/network-scripts/ifcfg-<网卡名>
# 修改如下配置

BOOTPROTO="static" # 使用静态IP地址，默认为dhcp 
IPADDR="192.168.6.31" # 设置的静态IP地址
NETMASK="255.255.255.0" # 子网掩码 
GATEWAY="192.168.6.1" # 网关地址 
DNS1="192.168.6.2" # DNS服务器（此设置没有用到，所以我的里面没有添加）

ONBOOT=yes  #设置网卡启动方式为 开机启动 并且可以通过系统服务管理器 systemctl 控制网卡

```

修改/etc/sysconfig/network

```
# Created by anaconda
NETWORKING=yes
GATEWAY=192.168.6.1
```

重启网络服务

```
service network restart
 ip a
```



系统初始化

```shell
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.6.20 k8s-master1
192.168.6.21 k8s-node1
192.168.6.22 k8s-node2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```
## 2、安装Docker/kubeadm/kubelet【所有节点】
这里我们使用docker作为容器引擎，也可以使用别的，比如：containerd
### 2.1 安装docker
```shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl enable docker && systemctl start docker
```
配置镜像下载加速器：
```shell
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://3fc19s4g.mirror.aliyuncs.com"]
}
EOF

systemctl restart docker
docker info
```
### 2.2 添加YUM软件源-阿里云
```shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 2.3 安装kubeadm/kubelet/kubectl
由于版本更新频繁，这里指定版本号部署
```shell
yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0
systemctl enable kubelet
```
## 3、部署kubernetes master 【master节点】
在master（192.168.6.20）节点上执行
```shell
kubeadm init \
  --apiserver-advertise-address=192.168.6.20 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.21.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all
```

- --apiserver-advertise-address 集群通讯地址
- --image-repository 由于默认拉取镜像是k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址
- --kubernetes-version k8s版本，与上面安装的一致
- --service-cidr 集群内部虚拟网络，pod统一访问
- --pod-network-cidr pod网络，与下面部署的CNI网络组件yaml中保持一致

**或者**使用配置文件引导
```shell
vi kubeadm.conf
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.21.0
imageRepository: registry.aliyuncs.com/google_containers 
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12 

kubeadm init --config kubeadm.conf --ignore-preflight-errors=all 
```
初始化完成后，记住最后输出的`kebeadm join`命令，其他节点加入时需要使用
```shell
kubeadm join 192.168.6.20:6443 --token o7dko2.n75ix5kucj6okqwx \
        --discovery-token-ca-cert-hash sha256:9dc850d6699c8991ae5c543993945f49ad5358733a2ceedadaea3360ed70fb30
```
拷贝kebectl使用的连接k8s认证文件到默认路径：
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
查看工作节点：
```shell
kubectl get nodes
NAME          STATUS     ROLES                  AGE    VERSION
k8s-master1   NotReady   control-plane,master   2m3s   v1.21.0
```

- 注：由于还没有部署网络插件，所以节点会显示为准备`NotReady`状态
- 参考资料：

[https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node)
​

## 4、部署kubernetes node 【node节点】
在node节点上执行（192.168.6.21/22）执行
向集群添加新节点，执行在`kubeadm init`输出的`kubeadm join`命令
```shell
kubeadm join 192.168.6.20:6443 --token o7dko2.n75ix5kucj6okqwx \
        --discovery-token-ca-cert-hash sha256:9dc850d6699c8991ae5c543993945f49ad5358733a2ceedadaea3360ed70fb30
```

- 注：默认token有效期是24小时，当过期之后，该token将不可用；这时需要创建新的token：
```shell
kubeadm token create --print-join-command
```

- 参考资料：[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)



## 5、部署网络插件（CNI）
Calico是一个纯三层的数据中心网络方案，是目前Kubernetes主流的网络方案。
下载Calico的YUM文件：
```shell
wget https://docs.projectcalico.org/manifests/calico.yaml
```
下载完成后还需要修改里面定义内容：

- pod网络（CALICO_IPV4POOL_CIDR），与前面`kubeadm init`中`--pod-network-cidr`指定一样
- 关闭ipip模式和修改typha_service_name 和修改replicas
calico网络，默认是ipip模式（在每台node主机创建一个tunl0网口，这个隧道链接所有的node容器网络，官网推荐不同的ip网段适合，比如aws的不同区域主机）

修改成BGP模式，它会以daemonset方式安装在所有node主机，每台主机启动一个bird(BGPclient)，它会将calico网络内的所有node分配的ip段告知集群内的主机，并通过本机的网卡eth0或者ens33转发数据

- **注：下面这里我们暂时只修改最后一步，pod网络（**CALICO_IPV4POOL_CIDR**）**
```shell
# 关闭ipip模式
- name: CALICO_IPV4POOL_IPIP
value: "off"

# 修改typha_service_name
typha_service_name: "calico-typha"

# 修改
  replicas: 1
  revisionHistoryLimit: 2

# 修改pod的网段CALICO_IPV4POOL_CIDR
- name: CALICO_IPV4POOL_CIDR
value: "10.244.0.0/16"
```
修改完成之后部署：
```shell
kubectl apply -f calico.yaml
kubectl get pods -n kube-system

# Calico Pod都为RUNNING之后，所有节点也会进入ready状态
```
### 问题处理
#### 5.1 CoreDNS问题处理：
```shell
kubectl get pods -n kube-system
NAME                                     READY   STATUS             RESTARTS   AGE
calico-kube-controllers-8db96c76-z7h5p   1/1     Running            0          16m
calico-node-pshdd                        1/1     Running            0          16m
calico-node-vjwlg                        1/1     Running            0          16m
coredns-545d6fc579-5hd9x                 0/1     ImagePullBackOff   0          16m
coredns-545d6fc579-wdbsz                 0/1     ImagePullBackOff   0          16m
```

- 可以通过`kubectl describe pod/coredns-545d6fc579-5hd9x  -n kube-system`进行查看，一般是镜像拉取问题

所有节点执行如下：
```shell
docker pull registry.aliyuncs.com/google_containers/coredns:1.8.0
docker tag registry.aliyuncs.com/google_containers/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
```

- 过一会儿，CoreDNS Pod会自动恢复正常。

注：以后所有yaml文件都只在Master节点执行。 
参考资料：
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
#### 5.2 某些Calico Pod无法启动
报错如下：
```shell
Readiness probe failed: caliconode is not ready: BIRD is not ready: BGP not established with
```
原因：在calico.yaml文件中，IP_AUTODETECTION_METHOD 配置项默认为first-found，这种模式中calico会使用第一获取到的有效网卡，虽然会排除docker网络，localhost啥的，但是在复杂网络环境下还是有出错的可能。在这次异常中主机上的calico选择了一个vbr网卡。
> 提供两种解决方案，第一种机器重启后依然生效，解决方案是修改calico.yaml中 IP_AUTODETECTION_METHOD 的默认值，第二种机器重启后需要重新配置，解决方案是使用calicoctl命令。

calico.yaml 文件添加以下二行：
```shell
- name: IP_AUTODETECTION_METHOD
  value: "interface=ens.*"  # ens 根据实际网卡开头配置，支持正则表达式
```
上下文如下：
```shell
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens.*"  # ens 根据实际网卡开头配置，支持正则表达式
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
            # Enable or Disable VXLAN on the default IP pool.
            - name: CALICO_IPV4POOL_VXLAN
              value: "Never"
```
更新部署并重启服务
```shell
kubectl apply -f calico.yaml
```
## 6、测试kubernetes集群
集群中创建一个nginx pod，验证是否正常运行
```shell
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
```
 	访问地址：http://NodeIP:Port
## 7、部署Dashboard
Dashboard是官方提供的一个UI，可用于基本管理K8s资源。
```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```


默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部：
```shell
vi recommended.yaml
...
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
...

kubectl apply -f recommended.yaml
kubectl get pods -n kubernetes-dashboard
```
	访问地址：[https://NodeIP:30001](https://NodeIP:30001)
创建service account 并绑定到默认的cluster-admin管理员集群角色：
```shell
# 创建用户
$ kubectl create serviceaccount dashboard-admin -n kube-system
# 用户授权 
$ kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 获取用户Token
$ kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```
注：使用输出的token登陆dashboard
## 8、切换容器引擎为Containerd
参考资料：[https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd)
### 8.1 配置先决条件
```shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置必需的 sysctl 参数，这些参数在重新启动后仍然存在。
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
### 8.2 安装Containerd
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install -y containerd.io
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
systemctl restart containerd
```
### 8.3 修改配置文件

- pause 镜像设置过阿里云镜像仓库地址
- cgroups 驱动设置为systemd
- 拉取Docker Hub 镜像配置加速地址设置为阿里云镜像仓库地址
```shell
vi /etc/containerd/config.toml
   [plugins."io.containerd.grpc.v1.cri"]
      sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"  
         ...
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
             SystemdCgroup = true
             ...
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://b9pmyelo.mirror.aliyuncs.com"]
          
systemctl restart containerd
```
### 8.4 配置kubelet使用Containerd
```shell
vi /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd

systemctl restart kubelet
```
### 8.5 验证
```shell
kubectl get node -o wide

k8s-node1  xxx  containerd://1.4.4
```
### 8.6 容器管理工具
Containerd提供了`ctrl`命令行工具管理容器，但功能比较简单，所以一般会用`crictl`工具验查和调试容器。
项目地址：[https://github.com/kubernetes-sigs/cri-tools/](https://github.com/kubernetes-sigs/cri-tools/)
设置crictl连接containerd：
```shell
vi /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```
#### docker和crictl命令对照表：
**镜像相关功能**

| **镜像相关功能** | **Docker** | **Containerd** |
| --- | --- | --- |
| 显示本地镜像列表 | docker images | crictl images |
| 下载镜像 | docker pull | crictl pull |
| 上传镜像 | docker push | 无，例如buildk |
| 删除本地镜像 | docker rmi | crictl rmi |
| 查看镜像详情 | docker inspect IMAGE-ID | crictl inspecti IMAGE-ID |

**容器相关功能**

| **容器相关功能** | **Docker** | **Containerd** |
| --- | --- | --- |
| 显示容器列表 | docker ps | crictl ps |
| 创建容器 | docker create | crictl create |
| 启动容器 | docker start | crictl start |
| 停止容器 | docker stop | crictl stop |
| 删除容器 | docker rm | crictl rm |
| 查看容器详情 | docker inspect | crictl inspect |
| attach | docker attach | crictl attach |
| exec | docker exec | crictl exec |
| logs | docker logs | crictl logs |
| stats | docker stats | crictl stats |

**pod相关功能**

| **POD相关功能** | **Docker** | **Containerd** |
| --- | --- | --- |
| 显示 POD 列表 | 无 | crictl pods |
| 查看 POD 详情 | 无 | crictl inspectp |
| 运行 POD | 无 | crictl runp |
| 停止 POD | 无 | crictl stopp |

