# 1. 首页

- 平台管理员首页：

![image-20251009090434039](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009090434039.png)

支持业务空间创建、查看和删除

![image-20251009090845087](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009090845087.png)

![image-20251009090856919](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009090856919.png)

![image-20251009090913477](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009090913477.png)

支持用户创建、查看、编辑、查看用户权限以及删除用户

![image-20251009090611408](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009090611408.png)

![image-20251009091111790](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091111790.png)

![image-20251009091134345](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091134345.png)

![image-20251009091148744](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091148744.png)

![image-20251009091201521](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091201521.png)

![image-20251009091214455](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091214455.png)

# 2. 工作空间

平台管理员或工作空间管理员支持在对应工作空间下创建命名空间以及邀请用户，并具备管理这些资源的权限

![image-20251009091415866](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091415866.png)

![image-20251009091442014](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091442014.png)

# 3. 命名空间

命名空间下可以查看所有改命名空间下的角色和服务账号信息（资源配额、成员暂未开发）

![image-20251009091525538](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009091525538.png)



# 4. 工作负载

工作负载是对整个k8s原生资源（deployment、statefulset、daemonset等）的可视化界面管控

## 4.1 有状态服务（deployment）

支持根据工作空间和命名空间的过滤，以及搜索和状态筛选、批量删除

![image-20251009092249404](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092249404.png)

创建deployment支持表单和YAML两种模式，表单的数据会自动解析到YAML中，避免用户直接接触复杂的YAML格式，简化用户学习使用k8s成本：

![image-20251009092427629](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092427629.png)

![image-20251009092601518](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092601518.png)

![image-20251009092613984](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092613984.png)



编辑：

![image-20251009092844931](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092844931.png)

支持多种挂载方式

![image-20251009092911114](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092911114.png)

![image-20251009092936229](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092936229.png)

删除：

![image-20251009092954047](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092954047.png)

## 4.2 有状态服务（statusfulset）

有状态服务的各功能通上deployment类似

![image-20251009093115630](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093115630.png)

## 4.3 守护进程（daemonset）

守护进程的各功能通上deployment类似

![image-20251009093138368](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093138368.png)

## 4.4 服务（service）

服务的各功能通上deployment类似

![image-20251009093304800](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093304800.png)



## 4.6 任务（job）

任务的各功能通上deployment类似

![image-20251009093419132](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093419132.png)

## 4.5 定时任务（cronjob）

定时任务的各功能通上deployment类似

![image-20251009093430198](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093430198.png)

定时任务支持对任务的暂停等功能

![image-20251009093517462](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009093517462.png)

# 容器组（pod）

批量删除

![image-20251009092717036](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009092717036.png)

容器组日志查看，支持查看指定容器、下载日志以及刷新日志，默认显示1000条数据：

![image-20251009095205919](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009095205919.png)



# 5. 配置文件

## 5.1 配置字典（configmap）

![image-20251009101800815](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009101800815.png)

支持创建、编辑、删除等功能

![image-20251009101913323](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009101913323.png)

![image-20251009101929436](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009101929436.png)

![image-20251009101942366](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009101942366.png)

编辑

![image-20251009102007689](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102007689.png)

删除

![image-20251009102026197](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102026197.png)

## 5.2 保密字典（secret）

同配置字典configmap功能类似，创建时支持多种类型

![image-20251009102132624](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102132624.png)

![image-20251009102158778](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102158778.png)

![image-20251009102209150](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102209150.png)

![image-20251009102227391](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102227391.png)



# 6. 存储和持久化

## 6.1 存储类（storageclass）

![image-20251009102341428](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102341428.png)



## 6.2 持久卷（pv）

![image-20251009102419179](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102419179.png)

## 6.3 持久卷申明（pvc）

支持创建和删除

![image-20251009102428780](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102428780.png)

![image-20251009102454554](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102454554.png)



# 7. 节点管理

平台管理员具备节点的启停、批量启停，编辑污点和label的功能

![image-20251009102536477](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102536477.png)

![image-20251009102707493](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102707493.png)

![image-20251009102717683](/images/posts/k8s可视化管理平台kubeants.assets/image-20251009102717683.png)

# # 8. k8s可视化管理平台kubeants部署清单(暂未整理完成)

- kubeants-controller

```
---
apiVersion: rbac.kubeants.io/v1beta1
kind: RoleTemplate
metadata:
  name: admin
spec:
  autoApply: true  # 是否自动应用到新建的 namespace
  namespaces: ["*"]  # 适用于所有 namespace
  # excludedNamespaces: ["kube-system"]  
  rules:
    - apiGroups: ["*"]  
      resources: ["*"]
      verbs: ["*"]
---
apiVersion: rbac.kubeants.io/v1beta1
kind: RoleTemplate
metadata:
  name: edit
spec:
  autoApply: true  
  namespaces: ["*"]  
  # namespaces: ["defualt", "kube-public", "kube-system"]  
  # namespaceSelector:
  #   matchLabels:
  #     kubeants.io/workspace: ws1
  # excludedNamespaces: ["kube-system"]  
  rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.kubeants.io/v1beta1
kind: RoleTemplate
metadata:
  name: view
spec:
  autoApply: true 
  namespaces: ["*"]  
  # excludedNamespaces: ["kube-system"]  
  rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["get", "list", "watch"]
---
apiVersion: user.kubeants.io/v1beta1
kind: User
metadata:
  name: admin
spec:
  name: "admin"
  email: "admin@ka.io"
  phone: "138xxxxxx"
  password: "Root@123"
  state: "active"
---
apiVersion: workspace.kubeants.io/v1beta1
kind: Workspace
metadata:
  name: "system-workspace"
spec:
  clusters: ["k8s1"]
---
apiVersion: workspace.kubeants.io/v1beta1
kind: Workspace
metadata:
  name: "ws1"
spec:
  clusters: ["k8s1"]
```



- kubeants-apiserver

```yaml
---
# 需要挂载当前k8s的kubeconfig
# kubectl -n kubeants-system create secret generic  kubeconfig --from-file=config=.kube/config
# apiVersion: v1
# data:
#   config: YXBpVmVyc2l...
# kind: Secret
# metadata:
#   name: kubeconfig
#   namespace: kubeants-system
# type: Opaque
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeants-apiserver
  namespace: kubeants-system
data:
  config.yaml: |-
    system:
      port: ":8080"
    jwt:
      secret: "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiSm9obiIsImFkbWluIjp0cnVlfQ.n-qvhsRi6C0zBGcKrMGv-qSZGUUssXTTbICvENxqDCjCL2ejSt62uTwemHZe4pLI_sNSr7FWEI2MKlqequemeg"  # 建议只保存密钥，不要存 token 全文
      expiration: 7200  # 单位：秒
    log:
      level: "debug"    # 支持：debug、info、warn、error
      format: "console" # 可选：console（开发）或 json（生产）
      file: ""          # 可选：写入文件的路径（默认空，即只输出到控制台）
    cors:
      enable: true
      allowedOrigins:   # 根据具体需求自行定义
        - "http://localhost:9528"
        - "http://localhost:8080"
        - "http://localhost"
        - "http://127.0.0.1:8080"
        - "http://kubeants-apiserver.kubeants-system"
        - "http://172.17.142.147:30001"
        - "http://172.17.142.147:30002"
      defaultOrigins: "*"  # 默认允许的 Origin，建议开发环境使用 localhost
      accessControlAllowCredentials: "true"
      accessControlAllowHeaders: "Content-Type,AccessToken,X-CSRF-Token, Authorization, Token,X-Token,X-User-Id"
      accessControlAllowMethods: "POST, GET, OPTIONS, DELETE, PUT, PATCH"
      accessControlExposeHeaders: "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers, Content-Type, New-Token, New-Expires-At"
    authz:
      exemptResources:
        - apiGroups: [""]
          versions: ["v1"]
          resources: ["namespaces", "persistentvolumes"]
          verbs: ["get", "list", "create"]
        - apiGroups: ["rbac.kubeants.io"]
          resources: ["roletemplates"]
          verbs: ["get", "list"]
        - apiGroups: ["storage.k8s.io"]
          versions: ["v1"]
          resources: ["storageclasses"]
          verbs: ["get", "list", "create"]
        - apiGroups: ["workspace.kubeants.io"]
          resources: ["*"]
          verbs: ["get", "list"]
        - apiGroups: ["userbinding.kubeants.io"]
          versions: ["v1beta1"]
          resources: ["userbindings"]
          verbs: ["get", "list", "delete", "create"]
        - apiGroups: ["user.kubeants.io"]
          versions: ["v1beta1"]      # 如果有版本号要加上
          resources: ["users"]
          verbs: ["get", "list", "create"]
        - apiGroups: ["rbac.kubeants.io"]
          resources: ["roletemplates"]
          verbs: ["get", "list"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeants-apiserver
  name: kubeants-apiserver
  namespace: kubeants-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeants-apiserver
  template:
    metadata:
      labels:
        app: kubeants-apiserver
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/geray/kubeants-apiserver:v1.7.15
        imagePullPolicy: IfNotPresent
        name: kubeants-apiserver
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /root/.kube/config
          name: kubeconfig-volume
          subPath: config
        - mountPath: /config.yaml
          name: kubeants-apiserver
          subPath: config.yaml
      dnsPolicy: ClusterFirst
      hostAliases:
      - hostnames:
        - lb.kubesphere.local
        ip: 172.17.142.147
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        runAsGroup: 0
        runAsUser: 0
      tolerations:
      - effect: NoSchedule
        key: node.kubernetes.io/memory-pressure
        operator: Exists
      volumes:
      - name: kubeconfig-volume
        secret:
          defaultMode: 420
          secretName: kubeconfig
      - configMap:
          defaultMode: 420
          name: kubeants-apiserver
        name: kubeants-apiserver
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kubeants-apiserver
  name: kubeants-apiserver
  namespace: kubeants-system
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
      nodePort: 30001
  selector:
    app: kubeants-apiserver
  type: NodePort
```







