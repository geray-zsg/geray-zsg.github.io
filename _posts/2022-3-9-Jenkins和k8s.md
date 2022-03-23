---
layout: post
title: "Jenkins基础（三）- Jenkins+k8s+SpringCloud持续集成"
description: "分享"
tag: Jenkins
---

# k8s构建Jenkins持续集成（一）

## 1、Jenkins的Master-Slave分布式构建

### 1. 什么是Master-Slave分布式构建

![image-20220317104318706](/images/posts/2022-3-9-Jenkins和k8s/image-20220317104318706.png)

Jenkins的Master-Slave分布式构建，就是通过将构建过程分配到从属Slave节点上，从而减轻Master节 点的压力，而且可以同时构建多个，有点类似负载均衡的概念。

### 2. 如何实现Master-Slave分布式构建

#### 1）开启代理程序的TCP端口

> Manage Jenkins -> Configure Global Security

![image-20220317111734380](/images/posts/2022-3-9-Jenkins和k8s/image-20220317111734380.png)



#### 2）新建节点，并连接到Master节点上

> Manage Jenkins—Manage Nodes—新建节点

![image-20220317112128992](/images/posts/2022-3-9-Jenkins和k8s/image-20220317112128992.png)

配置从节点

![image-20220317113143041](/images/posts/2022-3-9-Jenkins和k8s/image-20220317113143041.png)

有两种在Slave节点连接Master节点的方法

![image-20220317113357718](/images/posts/2022-3-9-Jenkins和k8s/image-20220317113357718.png)

推荐使用第二种

> 下载agent.jar，并上传到Slave节点，然后执行页面提示的命令

#### 3）测试节点是否可用

- 自由风格和Maven风格的项目：

![image-20220317113559022](/images/posts/2022-3-9-Jenkins和k8s/image-20220317113559022.png)

- 流水线风格的项目：

```
node('slave1') {
	stage('check out') {
		checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],	userRemoteConfigs: [[credentialsId: '68f2087f-a034-4d39-a9ff-1f776dd3dfa8', url:  git@192.168.6.61:itheima_group/tensquare_back_cluster.git']]])
	}
}
```

## 2、Kubernetes实现Master-Slave分布式构建方案

### 1. 传统Jenkins的Master-Slave方案的缺陷

- Master节点发生单点故障时，整个流程都不可用了 
- 每个 Slave节点的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导 致管理起来非常不方便，维护起来也是比较费劲 
- 资源分配不均衡，有的 Slave节点要运行的job出现排队等待，而有的Slave节点处于空闲状态 
- 资源浪费，每台 Slave节点可能是实体机或者VM，当Slave节点处于空闲状态时，也不会完全释放 掉资源

<u>以上种种问题，我们可以引入Kubernates来解决！</u>



### 2. Kubernates简介

Kubernetes（简称，K8S）是Google开源的容器集群管理系统，在Docker技术的基础上，为容器化的 应用提供部署运行、资源调度、服务发现和动态伸缩等一系列完整功能，提高了大规模容器集群管理的 便捷性。 其主要功能如下： 

- 使用Docker对应用程序包装(package)、实例化(instantiate)、运行(run)。 
- 以集群的方式运行、管理跨机器的容器。以集群的方式运行、管理跨机器的容器。 
- 解决Docker跨机器容器之间的通讯问题。解决Docker跨机器容器之间的通讯问题。 
- Kubernetes的自我修复机制使得容器集群总是运行在用户期望的状态。

### 3. Kubernates+Docker+Jenkins持续集成架构图

![image-20220317115920937](/images/posts/2022-3-9-Jenkins和k8s/image-20220317115920937.png)

![image-20220317144038835](/images/posts/2022-3-9-Jenkins和k8s/image-20220317144038835.png)

大致工作流程：

> 手动/自动构建 ---> Jenkins 调度 K8S API ---＞动态生成 Jenkins Slave pod --------＞ Slave pod 拉取 Git 代码／编译／打包镜像 ----＞推送到镜像仓库 Harbor --------＞ Slave 工作完成，Pod 自动销毁 ------＞部署 到测试或生产 Kubernetes平台。（完全自动化，无需人工干预）

### 4. Kubernates+Docker+Jenkins持续集成方案好处

#### 1）kubernetes架构

- 服务高可用：当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。 
- 动态伸缩，合理使用资源：每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。 
- 扩展性好：当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。



![image-20220317143923723](/images/posts/2022-3-9-Jenkins和k8s/image-20220317143923723.png)

- API Server：用于暴露Kubernetes API，任何资源的请求的调用操作都是通过kube-apiserver提供的接 口进行的。 
- Etcd：是Kubernetes提供默认的存储系统，保存所有集群数据，使用时需要为etcd数据提供备份计 划。 
- Controller-Manager：作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点 （Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额 （ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化 修复流程，确保集群始终处于预期的工作状态。 
- Scheduler：监视新创建没有分配到Node的Pod，为Pod选择一个Node。 
- Kubelet：负责维护容器的生命周期，同时负责Volume和网络的管理 
- Kube proxy：是Kubernetes的核心组件，部署在每个Node节点上，它是实现Kubernetes Service的通 信与负载均衡机制的重要组件。

#### 2）kubernetes部署以及环境介绍

有关kubernetes的搭建和知识请参考一下两个连接：

[云原生知识笔记](https://geray-zsg.github.io/tags/#%E4%BA%91%E5%8E%9F%E7%94%9F-ref)

[个人博客相关Kubernetes文档](https://geray-zsg.github.io/tags/#Kubernetes-ref)

| 主机名称         | IP地址       | 软件                                                         |
| ---------------- | ------------ | ------------------------------------------------------------ |
| 代码服务器       | 192.168.6.61 | Gitlab                                                       |
| Docker仓库服务器 | 192.168.6.62 | Harbor                                                       |
| k8s-master1      | 192.168.6.20 | kube-apiserver、kube-controller-manager、kubescheduler、docker、etcd、calico，NFS |
| k8s-node1        | 192.168.6.21 | kubelet、kubeproxy、Docker                                   |
| k8s-node2        | 192.168.6.22 | kubelet、kubeproxy、Docker                                   |



# k8s构建Jenkins持续集成（二）

## 1、Jenkins-Master-Slave架构图：

![image-20220317145210826](/images/posts/2022-3-9-Jenkins和k8s/image-20220317145210826.png)

## 2、安装和配置NFS文件服务器

### 1. NFS简介

NFS（Network File System），它最大的功能就是可以通过网络，让不同的机器、不同的操作系统可以 共享彼此的文件。我们可以利用NFS共享Jenkins运行的配置文件、Maven的仓库依赖文件等

### 2. NFS安装

我们把NFS服务器安装在192.168.6.20（k8s-master1）机器上

#### 1）安装NFS服务（在所有K8S的节点都需要安装）

```
yum install -y nfs-utils
```

#### 2）创建共享目录

```
mkdir -p /opt/nfs/jenkins

# 编写NFS的共享配置
vi /etc/exports 
# 内容如下: *代表对所有IP都开放此目录，rw是读写
/opt/nfs/jenkins *(rw,no_root_squash) 
```

#### 3）启动服务

```
systemctl enable nfs
systemctl start nfs 
```

#### 4）查看NFS共享目录

```
showmount -e 192.168.6.20
```

#### 5）测试

```
# 手动挂载
mount -t nfs 192.168.6.20:/opt/nfs/jenkins /var/testNFS

# 创建测试文件并查看
echo "nfs test1..." >> /opt/nfs/jenkins/test1.html

echo "nfs test2..." >> /var/testNFS/test2.html

# 卸载挂载
umount /var/testNFS
# 查看挂载
```



## 3、在Kubernetes安装Jenkins-Master（Jenkins主节点）

### 1. 创建NFS client provisioner

nfs-client-provisioner 是一个Kubernetes的简易NFS的外部provisioner，本身不提供NFS，需要现有 的NFS服务器提供存储。

#### 1）上传nfs-client-provisioner构建文件

- 其中注意修改deployment.yaml，使用之前配置NFS服务器和目录

#### 2）构建nfs-client-provisioner的pod资源

- deployment.yaml

```
cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
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
          image: lizhenliang/nfs-subdir-external-provisioner:v4.0.1
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.6.20 # nfs服务地址
            - name: NFS_PATH
              value: /opt/nfs/jenkins/ # 可挂载路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.6.20 # nfs服务地址
            path: /opt/nfs/jenkins/ # 可挂载路径
```

- class.yaml

```
cat class.yaml 
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # 必须和 deployment中的环境变量 env PROVISIONER_NAME' 保持一致
parameters:
  archiveOnDelete: "false"
```

- rbac.yaml

```
cat rbac.yaml 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
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
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```



```
# 将以上三个文件放入nfs-client中并部署
cd nfs-client
kubectl create -f .
```

#### 3）查看pod是否创建成功

```
kubeclt get pods
kubectl get -f .
kubectl get sc
kubectl describe sc managed-nfs-storage
```

![image-20220318100339377](/images/posts/2022-3-9-Jenkins和k8s/image-20220318100339377.png)

![image-20220318120007276](/images/posts/2022-3-9-Jenkins和k8s/image-20220318120007276.png)



### 2. 安装Jenkins-Master

#### 1）上传Jenkins-Master构建文件

1. 在StatefulSet.yaml文件，声明了利用nfs-client-provisioner进行Jenkins-Master文件存储

2. Service发布方法采用NodePort，会随机产生节点访问端口

#### 2）创建kube-ops的namespace

因为我们把Jenkins-Master的pod放到kube-ops名称空间下

```
kubectl create namespace kube-ops
```

#### 3）构建Jenkins-Master的pod资源

- StatefulSet.yaml 有状态服务

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  labels:
    name: jenkins
  namespace: kube-ops
spec:
  serviceName: jenkins
  selector:
    matchLabels:
      app: jenkins
  replicas: 1
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      name: jenkins
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-alpine
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 8080
            name: web
            protocol: TCP
          - containerPort: 50000
            name: agent
            protocol: TCP
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 0.5
              memory: 500Mi
          env:
            - name: LIMITS_MEMORY
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
                  divisor: 1Mi
            - name: JAVA_OPTS
              value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: jenkins-home
    spec:
      storageClassName: "managed-nfs-storage"
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

- Service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent

```

- ServiceAccount.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: kube-ops
```

- rbac.yaml

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
  namespace: kube-ops
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods", "events"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: kube-ops
    
---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkinsClusterRole
  namespace: kube-ops
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkinsClusterRuleBinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkinsClusterRole
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: kube-ops
```

> 需要添加events事件操作，否则构建时会把如下错误

![image-20220322113930149](/images/posts/2022-3-9-Jenkins和k8s/image-20220322113930149.png)

```
# 将以上资源放置到jenkins-master目录
cd jenkins-master
kubectl create -f .
```

#### 4）查看pod是否创建成功

```
[root@k8s-master1 nfs-test]# kubectl get pod -n kube-ops
NAME        READY   STATUS    RESTARTS   AGE
jenkins-0   1/1     Running   0          107m
```

#### 5）查看信息，并访问

查看Pod运行在那个Node上

```
kubectl describe pods -n kube-ops
```

查看分配的端口

```
kubectl get service -n kube-ops
```

#### 6）修改为国内地址（清华大学）

Jenkins国外官方插件地址下载速度非常慢，所以可以修改为国内插件地址：

> 清华大学：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/
>
> 阿里云：https://mirrors.aliyun.com/jenkins/updates/
>
> 华为云：https://mirrors.huaweicloud.com/jenkins/updates/

直接复制update-center.json即可



#### 7）先安装基本的插件

- Localization:Chinese 
- Git 
- Pipeline 
- Extended Choice Parameter

## 4、Jenkins（主节点）与Kubernetes整合

### 1. 安装Kubernetes插件

![image-20220318133633622](/images/posts/2022-3-9-Jenkins和k8s/image-20220318133633622.png)

### 2. 实现Jenkins与Kubernetes整合

> 系统管理->系统配置->云->新建云->Kubernetes

![image-20220318140131922](/images/posts/2022-3-9-Jenkins和k8s/image-20220318140131922.png)

- kubernetes地址：采用了k8s的服务器发现：https://kubernetes.default.svc.cluster.local （基本是固定的地址）
- k8s的命名空间（namespace）：填kube-ops（Jenkins的安装的命名空间），然后点击Test Connection，如果出现 Connection test successful 的提示信息证明 Jenkins 已经可以和 Kubernetes 系统正常通信
- Jenkins URL 地址：http://jenkins.kube-ops.svc.cluster.local:8080（基本也是固定的）

## 5、构建Jenkins-Slave自定义镜像（Jenkins从节点）

Jenkins-Master在构建Job的时候，Kubernetes会创建Jenkins-Slave的Pod来完成Job的构建。我们使用官方的镜像为官方推荐镜像：jenkins/jnlp-slave:latest，但是这个镜像里面并没有Maven 环境，为了方便使用，我们需要自定义一个新的镜像：

### 构建自己的Maven环境的镜像

**准备材料：**

| 文件名                        | 说明          |
| ----------------------------- | ------------- |
| Dockerfile                    | 构建镜像文件  |
| apache-maven-3.6.2-bin.tar.gz | Maven环境     |
| settings.xml                  | Maven配置文件 |

Dockerfile：

```
FROM jenkins/jnlp-slave:latest

MAINTAINER itcast

# 切换到 root 账户进行操作
USER root

# 安装 maven
COPY apache-maven-3.6.2-bin.tar.gz .

RUN tar -zxf apache-maven-3.6.2-bin.tar.gz && \
    mv apache-maven-3.6.2 /usr/local && \
    rm -f apache-maven-3.6.2-bin.tar.gz && \
    ln -s /usr/local/apache-maven-3.6.2/bin/mvn /usr/bin/mvn && \
    ln -s /usr/local/apache-maven-3.6.2 /usr/local/apache-maven && \
    mkdir -p /usr/local/apache-maven/repo

COPY settings.xml /usr/local/apache-maven/conf/settings.xml

# USER jenkins
```

settings.xml：

> 修改仓库位置和私服

```
  <!-- 仓库设置 -->
  <localRepository>/usr/local/apache-maven/repo</localRepository>


  <mirrors>
   <!-- 配置具体的仓库的下载镜像 -->
    <mirror>
      <!-- 此镜像的唯一标识，用来区分不同的mirrors元素 -->
      <id>nexus-aliyun</id>
      <!-- 被替代仓库的名称（对应上面查找到的默认仓库id） -->
      <mirrorOf>central</mirrorOf>
      <!-- 镜像名称，随意 -->
      <name>Nexus aliyun</name>
      <!-- 镜像URL -->
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
  </mirrors>
```

**镜像构建：**

构建出一个新镜像：jenkins-slave-maven:latest 

```
docker build -t geray/jenkins-slave-maven:latest .
```

**推送到Harbor仓库：**

```
docker tag geray/jenkins-slave-maven:latest 192.168.6.62:85/library/jenkins-slave-maven:latest

docker push 192.168.6.62:85/library/jenkins-slave-maven:latest
```



## 6、测试Jenkins-Slave是否可以创建

### 1. 创建一个Jenkins流水线项目

> 名为test_jenkins_slave



### 2. 编写Pipeline，从GItlab拉取代码

```
// 项目拉去方式
def git_address = "http://192.168.6.61:82/geray_group/tensquare_back.git"
// 拉取代码所使用的凭证
def git_auth = "25812798-73cf-4f59-b102-1da677a7f90c"

//创建一个Pod的模板，label为jenkins-slave
podTemplate(label: 'jenkins-slave', cloud: 'kubernetes', containers: [
		containerTemplate(
			name: 'jnlp',
			image: "192.168.6.62:85/library/jenkins-slave-maven:latest"
		)
	]
)

{
	//引用jenkins-slave的pod模块来构建Jenkins-Slave的pod
	node("jenkins-slave"){
	// 第一步
		stage('拉取代码'){
			checkout([$class: 'GitSCM', branches: [[name: 'master']],
			userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]])
		}
	}
}
```

### 3. 查看构建日志（观察）



> 在构建的同时会创建一个Jenkins的节点，并在完成之后回收该节点（为master分摊压力）

![image-20220318150916597](/images/posts/2022-3-9-Jenkins和k8s/image-20220318150916597.png)

![image-20220318150847592](/images/posts/2022-3-9-Jenkins和k8s/image-20220318150847592.png)



## 7、Jenkins+Kubernetes+Docker完成微服务持续集成

### 1. 拉取代码，构建镜像

#### 1）创建NFS共享目录

让所有Jenkins-Slave构建指向NFS的Maven的共享仓库目录

```
mkdir /opt/nfs/maven

vi /etc/exports
# 添加内容：
/opt/nfs/jenkins *(rw,no_root_squash)
/opt/nfs/maven *(rw,no_root_squash)

# 重启NFS
systemctl restart nfs 

# 验证
showmount -e 192.168.6.20
```

#### 2）创建流水线项目（tensquare_back），编写构建Pipeline

> 添加Harbor仓库凭证（用户名密码）

> 构建项目的分支（字符参数）：branch
>
> 构建项目的名称（多选框）：project_name 



![image-20220318172138787](/images/posts/2022-3-9-Jenkins和k8s/image-20220318172138787.png)

```
// 项目拉去方式
def git_address = "http://192.168.6.61:82/geray_group/tensquare_back.git"
// 拉取代码所使用的凭证
def git_auth = "25812798-73cf-4f59-b102-1da677a7f90c"

//构建版本的名称
def tag = "latest"
//Harbor私服地址
def harbor_url = "192.168.6.62:85"
//Harbor的项目名称
def harbor_project_name = "tensquare"

//Harbor的凭证
def harbor_auth = "bb0cf66d-86fe-4aac-8862-4cc1864eda7e"

// k8s的pod模板
podTemplate(label: 'jenkins-slave', cloud: 'kubernetes', containers: [
        containerTemplate(
            name: 'jnlp',
            image: "192.168.6.62:85/library/jenkins-slave-maven:latest"
        ),
        // 引入docker环境，后面docker命令需要
        containerTemplate(
            name: 'docker',
            image: "docker:stable",
            ttyEnabled: true,
            command: 'cat'
        ),
	],
    volumes: [
    	// 将本地docker环境映射到pod中
    	hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    	// 挂在maven仓库到nfs文件共享服务器，保证数据不会随着pod的回收被丢失
        nfsVolume(mountPath: '/usr/local/apache-maven/repo', serverAddress: '192.168.6.20', serverPath: '/opt/nfs/maven'),
    ],
)

{
    node("jenkins-slave"){
        // 第一步
        stage('拉取代码'){
            checkout([$class: 'GitSCM', branches: [[name: '${branch}']], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]])
        }
        
    	// 第二步
        stage('编译并安装公共工程代码'){
            //编译并安装公共工程
            sh "mvn -f tensquare_common clean install"
        }
        
        // 第三步
        stage('构建镜像，部署项目'){
       		//把选择的项目信息转为数组
        	def selectedProjects = "${project_name}".split(',')
       		for(int i=0;i<selectedProjects.size();i++){
        		//取出每个项目的名称和端口
        		def currentProject = selectedProjects[i];
                //项目名称
                def currentProjectName = currentProject.split('@')[0]
                //项目启动端口
                def currentProjectPort = currentProject.split('@')[1]
                
                //定义镜像名称
                def imageName = "${currentProjectName}:${tag}"
                
                //编译，构建本地镜像
                sh "mvn -f ${currentProjectName} clean package dockerfile:build"
                
                // 引入docker环境给镜像打标签并上传到harbor仓库
                container('docker') {
                    //给镜像打标签
                    sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"
                    
                	//登录Harbor，并上传镜像（解析Harbor凭证并获取到相关账号密码）
                	withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]){
                        //登录
                        sh "docker login -u ${username} -p ${password} ${harbor_url}"
                        //上传镜像
                        sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"
                	}
                    //删除本地镜像
                    sh "docker rmi -f ${imageName}"
                    sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${imageName}"
                }
             }
        }
    }
}
```

#### 问题处理

![image-20220321185609183](/images/posts/2022-3-9-Jenkins和k8s/image-20220321185609183.png)

```
[Pipeline] { (构建镜像，部署项目)
[Pipeline] sh
+ mvn -f tensquare_eureka_server clean package dockerfile:build
[INFO] Scanning for projects...
[ERROR] [ERROR] Some problems were encountered while processing the POMs:
[ERROR] 'dependencies.dependency.version' for org.springframework.cloud:spring-cloud-starter-netflix-eureka-server:jar is missing. @ line 16, column 21
 @ 
[ERROR] The build could not read 1 project -> [Help 1]
[ERROR]   
[ERROR]   The project com.tensquare:tensquare_eureka_server:1.0-SNAPSHOT (/home/jenkins/agent/workspace/tensquare_back/tensquare_eureka_server/pom.xml) has 1 error
[ERROR]     'dependencies.dependency.version' for org.springframework.cloud:spring-cloud-starter-netflix-eureka-server:jar is missing. @ line 16, column 21
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
```

- pom.xml中没有设置eureka的版本导致，添加版本信息并提交

![image-20220321185716890](/images/posts/2022-3-9-Jenkins和k8s/image-20220321185716890.png)

#### 3）构建测试：

![image-20220321185513165](/images/posts/2022-3-9-Jenkins和k8s/image-20220321185513165.png)

可以看到harbor仓库已经成功被推送镜像

![image-20220322094309605](/images/posts/2022-3-9-Jenkins和k8s/image-20220322094309605.png)

### 2. 微服务部署到K8S

#### 1）安装Kubernetes Continuous Deploy插件



#### 2）建立k8s认证凭证

> 系统管理 --> Manage Credentials 

将k8s的`/root/.kube/config`文件拷贝到

![image-20220322095426717](/images/posts/2022-3-9-Jenkins和k8s/image-20220322095426717.png)

填入下图：

![image-20220322095719860](/images/posts/2022-3-9-Jenkins和k8s/image-20220322095719860.png)

#### 3）在每个项目下建立deploy.xml

```
---
apiVersion: v1
kind: Service
metadata:
  name: eureka
  labels:
    app: eureka
spec:
  type: NodePort
  ports:
    - port: 10086
      name: eureka
      targetPort: 10086
  selector:
    app: eureka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka
spec:
  serviceName: "eureka"
  replicas: 2
  selector:
    matchLabels:
      app: eureka
  template:
    metadata:
      labels:
        app: eureka
    spec:
      imagePullSecrets:
        - name: $SECRET_NAME
      containers:
        - name: eureka
          image: $IMAGE_NAME
          ports:
            - containerPort: 10086
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: EUREKA_SERVER
              value: "http://eureka-0.eureka:10086/eureka/,http://eureka-1.eureka:10086/eureka/"
            - name: EUREKA_INSTANCE_HOSTNAME
              value: ${MY_POD_NAME}.eureka
  podManagementPolicy: "Parallel"
```



#### 4）修改每个微服务的application.yml

Eureka：

```
server:
  port: ${PORT:10086}
spring:
  application:
    name: eureka

eureka:
  server:
    # 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）
    eviction-interval-timer-in-ms: 5000
    enable-self-preservation: false
    use-read-only-response-cache: false
  client:
    # eureka client间隔多久去拉取服务注册信息 默认30s
    registry-fetch-interval-seconds: 5
    serviceUrl:
      defaultZone: ${EUREKA_SERVER:http://127.0.0.1:${server.port}/eureka/}
  instance:
    # 心跳间隔时间，即发送一次心跳之后，多久在发起下一次（缺省为30s）
    lease-renewal-interval-in-seconds: 5
    # 在收到一次心跳之后，等待下一次心跳的空档时间，大于心跳间隔即可，即服务续约到期时间（缺省为90s）
    lease-expiration-duration-in-seconds: 10
    instance-id: ${EUREKA_INSTANCE_HOSTNAME:${spring.application.name}}:${server.port}@${random.long(1000000,9999999)}
    # EUREKA_INSTANCE_HOSTNAME 是引用了deploy.yaml中的环境变量
    hostname: ${EUREKA_INSTANCE_HOSTNAME:${spring.application.name}}
    # spring.application.name 是当前服务的名称，也就是上面的eureka
```

其他微服务需要注册到所有Eureka中:

```
# Eureka配置
eureka:
	client:
		serviceUrl:
			defaultZone: http://eureka-0.eureka:10086/eureka/,http://eureka-1.eureka:10086/eureka/ # Eureka访问地址
	instance:
		preferIpAddress: true
```



#### 5）修改后的流水线脚本

- 将一下流水线脚本添加到第6节的脚本中

```
// 最顶部定义k8s_auth的凭证
def k8s_auth = "b968a810-f183-4298-813e-d98a7d891fb0"

// 以下内容添加到删除本地镜像
               def deploy_image_name = "${harbor_url}/${harbor_project_name}/${imageName}"
                
                //部署到K8S
                sh """
                    sed -i 's#\$IMAGE_NAME#${deploy_image_name}#' ${currentProjectName}/deploy.yaml
                    sed -i 's#\$SECRET_NAME#${secret_name}#' ${currentProjectName}/deploy.yaml
                """
                
                kubernetesDeploy configs: "${currentProjectName}/deploy.yaml",
                kubeconfigId: "${k8s_auth}"
```



![image-20220322101827158](/images/posts/2022-3-9-Jenkins和k8s/image-20220322101827158.png)

添加部署步骤：

![image-20220322102049307](/images/posts/2022-3-9-Jenkins和k8s/image-20220322102049307.png)



#### 6）先测试eureka服务（问题处理）

```
[Pipeline] End of Pipeline
groovy.lang.MissingPropertyException: No such property: secret_name for class: groovy.lang.Binding
	at groovy.lang.Binding.getVariable(Binding.java:63)
	at org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SandboxInterceptor.onGetProperty(SandboxInterceptor.java:271)
```



![image-20220322104013784](/images/posts/2022-3-9-Jenkins和k8s/image-20220322104013784.png)

> 出现以上问题是因为没有给部署的服务传递harbor的秘钥信息，在deploy.yaml中需要harbor的秘钥信息

##### 创建harbor的凭据并在Jenkinsfile脚本中添加变量

> 证书的获取方法如下一小节`生成Docker凭证`

```
// 定义harbor的凭据
def secret_name = "registry-auth-secret"

// 这里的registry-auth-secret必须和k8s中的secret名称一至
```

> k8s会根据上面的定义将凭证信息传入deploy.yaml中并拉取镜像进行部署

#### 7）生成Docker凭证

```
# 1. 先登录私服
docker login -u geray -p Geray@123 192.168.6.62:85

# 2. 创建连接私服harbor的secret（生成证书信息） --- 邮箱信息可有可无
kubectl -n kube-ops create secret docker-registry registry-auth-secret --docker-server=192.168.6.62:85 --docker-username=geray --docker-password=Geray@123 --docker-email=geray@geray.cn

# 3. 查看信息
kubectl get secret -n kube-ops

```



#### 8）完整的Jenkinsfile文件

```
// 项目拉去方式
def git_address = "http://192.168.6.61:82/geray_group/tensquare_back.git"
// 拉取代码所使用的凭证
def git_auth = "25812798-73cf-4f59-b102-1da677a7f90c"

//构建版本的名称
def tag = "latest"
//Harbor私服地址
def harbor_url = "192.168.6.62:85"
//Harbor的项目名称
def harbor_project_name = "tensquare"

//Harbor的凭证
def harbor_auth = "bb0cf66d-86fe-4aac-8862-4cc1864eda7e"

// 最顶部定义k8s_auth的凭证
def k8s_auth = "b968a810-f183-4298-813e-d98a7d891fb0"

// 定义harbor的凭据
def secret_name = "registry-auth-secret"

// k8s的pod模板
podTemplate(label: 'jenkins-slave', cloud: 'kubernetes', containers: [
        containerTemplate(
            name: 'jnlp',
            image: "192.168.6.62:85/library/jenkins-slave-maven:latest",
            resourceRequestMemory: '1Gi',
            resourceLimitMemory: '1Gi',
			resourceRequestCpu: '1000m',
            resourceLimitCpu: '1000m'
        ),
        // 引入docker环境，后面docker命令需要
        containerTemplate(
            name: 'docker',
            image: "docker:stable",
            ttyEnabled: true,
            command: 'cat'
        ),
	],
    volumes: [
    	// 将本地docker环境映射到pod中
    	hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    	// 挂在maven仓库到nfs文件共享服务器，保证数据不会随着pod的回收被丢失
        nfsVolume(mountPath: '/usr/local/apache-maven/repo', serverAddress: '192.168.6.20', serverPath: '/opt/nfs/maven'),
    ],
)

{
    node("jenkins-slave"){
        // 第一步
        stage('拉取代码'){
            checkout([$class: 'GitSCM', branches: [[name: '${branch}']], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]])
        }
        
    	// 第二步
        stage('编译并安装公共工程代码'){
            //编译并安装公共工程
            sh "mvn -f tensquare_common clean install"
            sh "echo 安装成功！"
        }
        
        // 第三步
        stage('构建镜像，部署项目'){
       		//把选择的项目信息转为数组
        	def selectedProjects = "${project_name}".split(',')
       		for(int i=0;i<selectedProjects.size();i++){
        		//取出每个项目的名称和端口
        		def currentProject = selectedProjects[i];
                //项目名称
                def currentProjectName = currentProject.split('@')[0]
                //项目启动端口
                def currentProjectPort = currentProject.split('@')[1]
                
                //定义镜像名称
                def imageName = "${currentProjectName}:${tag}"
                
                //编译，构建本地镜像
                sh "mvn -f ${currentProjectName} clean package dockerfile:build"
                
                // 引入docker环境给镜像打标签并上传到harbor仓库
                container('docker') {
                    //给镜像打标签
                    sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"
                    
                	//登录Harbor，并上传镜像（解析Harbor凭证并获取到相关账号密码）
                	withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]){
                        //登录
                        sh "docker login -u ${username} -p ${password} ${harbor_url}"
                        //上传镜像
                        sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"
                	}
                    //删除本地镜像
                    sh "docker rmi -f ${imageName}"
                    sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${imageName}"
                }
            
               def deploy_image_name = "${harbor_url}/${harbor_project_name}/${imageName}"
                
                //部署到K8S
                sh """
                    sed -i 's#\$IMAGE_NAME#${deploy_image_name}#' ${currentProjectName}/deploy.yaml
                    sed -i 's#\$SECRET_NAME#${secret_name}#' ${currentProjectName}/deploy.yaml
                    cat ${currentProjectName}/deploy.yaml
                """
                
                kubernetesDeploy configs: "${currentProjectName}/deploy.yaml",
                kubeconfigId: "${k8s_auth}"
                
             }
        }
    }
}
```





#### 9）部署其他微服务（zuul）

zuul：deploy.yaml

```
---
apiVersion: v1
kind: Service
metadata:
  name: zuul
  labels:
    app: zuul
spec:
  type: NodePort
  ports:
    - port: 10020
      name: zuul
      targetPort: 10020
  selector:
    app: zuul
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zuul
spec:
  serviceName: "zuul"
  replicas: 2
  selector:
    matchLabels:
      app: zuul
  template:
    metadata:
      labels:
        app: zuul
    spec:
      imagePullSecrets:
        - name: $SECRET_NAME
      containers:
        - name: zuul
          image: $IMAGE_NAME
          ports:
            - containerPort: 10020
  podManagementPolicy: "Parallel"
```



#### 10）项目构建后，查看服务创建情况

![image-20220323110154093](/images/posts/2022-3-9-Jenkins和k8s/image-20220323110154093.png)

