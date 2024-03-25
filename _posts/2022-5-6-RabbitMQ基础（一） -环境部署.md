---
layout: post
title: "RabbitMQ基础（一）- 环境部署"
description: "分享"
tag: RabbitMQ
---

# 1、单节点安装

官网地址 [https://www.rabbitmq.com/download.html](https://www.rabbitmq.com/download.html)

## 1. 安装rabbitmq依赖erlang环境

配置[阿里云镜像加速](https://developer.aliyun.com/mirror/)

```shell
# net工具箱
yum install net-tools -y

# 启动EPEL源
sudo yum install epel-release -y

# 下载 erlang 安装包
wget https://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
 
# 解压
rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
 
# 安装
sudo yum install erlang -y

yum clean all

erl -version
```

> yum安装erlang很慢，修改`/etc/yum.conf`，打开本地缓存
>
> keepcache=1（默认为0）

## 2. 安装 socat

在`RabiitMQ`安装过程中需要依赖`socat`插件，首先安装该插件

```shell
yum install -y socat
```

> Socat 是 Linux 下的一个多功能的网络工具，名字来由是 「Socket CAT」。其功能与有瑞士军刀之称的 Netcat 类似，可以看做是 Netcat 的加强版。

## 3. 解压rabbitmq并修改配置文件

```shell
tar xf /opt/src/rabbitmq-server-generic-unix-3.10.0.tar.xz -C /opt/
mv rabbitmq_server-3.10.0/ rabbitmq
```

### 1）添加配置文件：rabbitmq-env.conf

```shell
cat >> /opt/rabbitmq/etc/rabbitmq/rabbitmq-env.conf << EOF
RABBITMQ_NODENAME=rabbitmq001
RABBITMQ_NODE_IP_ADDRESS=192.168.6.21
RABBITMQ_NODE_PORT=5672
RABBITMQ_MNESIA_BASE=/opt/rabbitmq/data
RABBITMQ_LOG_BASE=/opt/rabbitmq/logs
EOF
```

创建目录：

```shell
mkdir -p /opt/rabbitmq/{data,logs}
```

参数说明：

| 属性                     | 描述                                       | 默认值                                   |
| ------------------------ | ------------------------------------------ | ---------------------------------------- |
| RABBITMQ_NODENAME        | rabbitmq节点名称，集群中要注意节点名称唯一 | linux 默认节点名为 rabbit@$hostname      |
| RABBITMQ_NODE_IP_ADDRESS | 绑定的网络接口                             | 默认为空字符串表示绑定本机所有的网络接口 |
| RABBITMQ_NODE_PORT       | 端口                                       | 默认为5672                               |
| RABBITMQ_MNESIA_BASE     | mnesia所在路径                             | $RABBITMQ_HOME/var/lib/rabbitmq/mnesia   |
| RABBITMQ_LOG_BASE        | 日志所在路径                               | $RABBITMQ_HOME/var/log/rabbitmq          |

### 2）添加配置文件：rabbitmq.config

```
cat >> /opt/rabbitmq/etc/rabbitmq/rabbitmq.config <<EOF
[
 {rabbit,
   [
     {tcp_listeners, [5672]},
     {dump_log_write_threshold, [1000]},
     {vm_memory_high_watermark, 0.5},
     {disk_free_limit, "200MB"},
     {hipe_compile,true}
   ]
  }
].
EOF
```

> **注意：[]. 后面有一个点**

参数说明

| Key                      | Documentation                                                |
| ------------------------ | ------------------------------------------------------------ |
| tcp_listeners            | 用于监听 AMQP连接的端口列表(无SSL). 可以包含整数 (即”监听所有接口”)或者元组如 {“127.0.0.1”, 5672} 用于监听一个或多个接口.Default: [5672] |
| dump_log_write_threshold | 更改mnesia的转储日志写入阈值 Default: [100]                  |
| vm_memory_high_watermark | 流程控制触发的内存阀值．相看memory-based flow control 文档.Default: 0.4 |
| disk_free_limit          | RabbitMQ存储数据分区的可用磁盘空间限制．当可用空间值低于阀值时，流程控制将被触发.此值可根据RAM的总大小来相对设置 (如.{mem_relative, 1.0}).此值也可以设为整数(单位为bytes)或者使用数字单位(如．”50MB”).默认情况下，可用磁盘空间必须超过50MB.参考 Disk Alarms 文档.Default: 50000000 |
| hipe_compile             | 将此选项设置为true,将会使用HiPE预编译部分RabbitMQ,Erlang的即时编译器.这可以增加服务器吞吐量，但会增加服务器的启动时间． |


### 3）启动测试

```
# 启动测试
/opt/rabbitmq/sbin/rabbitmq-server start

# 后台启动
rabbitmq-server -detached

# 停止使用
rabbitmqctl stop
```

> 注意：出现completed with，表示启动成功

查看cookie文件是否存在

**注意：此文件必须存在**

> cookie文件中只保存了一行随机数，可以使用`cat`命令进行查看
>
> 默认的存储路径为 `/var/lib/rabbitmq/.erlang.cookie` 或` $HOME/.erlang.cookie`，该文件是一个隐藏文件，需要使用 `ls -al `命令查看。
>
> - 如果我们使用解压缩方式安装部署的rabbitmq，那么这个文件会在${home}目录下，也就是`$home/.erlang.cookie`。
> - 如果我们使用rpm等安装包方式进行安装的，那么这个文件会在/var/lib/rabbitmq目录下。

```
ls -a /home/rabbitmq/.erlang.cookie
```



## 4. 配置环境变量

```
echo 'export PATH=/opt/rabbitmq/sbin:$PATH' >> /etc/profile.d/my_env.sh

source /etc/profile
```

## 5. 启动web管理界面

```
rabbitmq-plugins enable rabbitmq_management
```

### 1）创建账号并授权访问

> guest的默认账号受限，默认只能localhost 浏览器访问管理界面，否则最初是登不上去的。
>

```shell
#添加vhost
rabbitmqctl add_vhost /testhost


#列出vhost
rabbitmqctl list_vhosts

#删除vhost
rabbitmqctl delete_vhost /testhost

# 创建账号和密码
rabbitmqctl add_user admin Geray_2022

# 设置用户角色
rabbitmqctl set_user_tags admin administrator

# 设置用户权限
# 用户 user_admin 具有/vhost1 这个 virtual host 中所有资源的配置、写、读权限
# 格式：set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
# 给/这个vhost设置了admin权限
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"

# 当前用户和角色
rabbitmqctl list_users
```

浏览器访问并使用新创建的用户进行登陆访问

http://192.168.6.21:15672/

## 6. 重置命令

```
# 关闭应用的命令
rabbitmqctl stop_app

# 清除的命令
rabbitmqctl reset

# 重新启动命令为
rabbitmqctl start_app
```



# 2、集群部署

## 1. 集群架构介绍

当单台 RabbitMQ 服务器的处理消息的能力达到瓶颈时，此时可以通过 RabbitMQ 集群来进行扩展，从而达到提升吞吐量的目的。RabbitMQ 集群是一个或多个节点的逻辑分组，集群中的每个节点都是对等的，每个节点共享所有的用户，虚拟主机，队列，交换器，绑定关系，运行时参数和其他分布式状态等信息。

| 主机名      | IP地址       | 部署服务                        | 操作系统   | 配置  |
| ----------- | ------------ | ------------------------------- | ---------- | ----- |
| k8s-master1 | 192.168.6.20 | RabbitMQ + HAProxy + KeepAlived | CentOS 7.9 | 4核4G |
| k8s-node1   | 192.168.6.21 | RabbitMQ + HAProxy + KeepAlived | CentOS 7.9 | 4核8G |
| k8s-node2   | 192.168.6.22 | RabbitMQ + HAProxy + KeepAlived | CentOS 7.9 | 4核4G |

## 2. 安装erlang、socat和rabbitmq

之前在`192.168.6.21`节点已经成功部署了一台单节点，所以只需要在另外两台服务上进行安装，安装方式请查看单节点安装方式。

> 注意：RABBITMQ_NODENAME和IP唯一

## 3. 环境初始化-配置域名解析

```
cat >> /etc/hosts <<EOF
192.168.6.20 k8s-master1
192.168.6.21 k8s-node1
192.168.6.22 k8s-node2
EOF
```

## 4. 配置 Erlang Cookie

将 `k8s-node1` 上的 `.erlang.cookie` 文件拷贝到其他两台主机上。该 `cookie` 文件相当于密钥令牌，集群中的 RabbitMQ 节点需要通过交换密钥令牌以获得相互认证，因此处于同一集群的所有节点需要具有相同的密钥令牌，否则在搭建过程中会出现 `Authentication Fail `错误。

RabbitMQ 服务启动时，`erlang VM` 会自动创建该` cookie` 文件，默认的存储路径为 `/var/lib/rabbitmq/.erlang.cookie` 或` $HOME/.erlang.cookie`，该文件是一个隐藏文件，需要使用 `ls -al `命令查看。（拷贝`.cookie`时，各节点都必须停止MQ服务）

```
# 停止rabbitmq的所有节点服务
rabbitmqctl stop

# 拷贝node1节点的.cookie文件到到其他两台服务
scp /root/.erlang.cookie root@k8s-master1:/root/
scp /root/.erlang.cookie root@k8s-node2:/root/

# 查看.cookie文件的权限和内容是否一致
cat /root/.erlang.cookie
ls -l /root/.erlang.cookie 
```

> 建议将 cookie 文件原来的 400 权限改为 600
>
> ```
> chmod 600 /root/.erlang.cookie
> ```
>

## 5. 启动服务并开启EPMD进程端口

```
su -s /bin/bash rabbitmq 
rabbitmq-server -detached

# 开启防火墙 4369 端口（直接关闭防火墙）
firewall-cmd --zone=public --add-port=4369/tcp --permanent
# 重启
systemctl restart firewalld.service
```

## 6. 集群搭建

RabbitMQ 集群的搭建需要选择其中任意一个节点为基准，将其它节点逐步加入。这里我们以 `k8s-node1 `为基准节点，将 `k8s-node2 `和 `k8s-master1`加入集群。在`k8s-node2 `和 `k8s-master1`上执行以下命令：

```
# 关闭应用的命令
rabbitmqctl stop_app

# 清除的命令（如果使用内存节点方式加入集群忽略这一步，并）
rabbitmqctl reset

# 3.节点加入, 在一个node加入cluster之前，必须先停止该node的rabbitmq应用，即先执行stop_app
# master1加入node1, node2加入node1
rabbitmqctl join_cluster rabbitmq1@k8s-node1

# 4.启动服务
rabbitmqctl start_app
```

> `join_cluster`加入命令中的`rabbitmq1@k8s-node1`可以通过`rabbitmqctl cluster_status | grep -i "Cluster name"`命令获取到
>
> ```
> [root@k8s-node1 opt]# rabbitmqctl cluster_status | grep -i "Cluster name"
> Cluster name: rabbitmq1@k8s-node1
> ```
>
> 也可以修改集群名称：
>
> ```
> rabbitmqctl set_cluster_name my_rabbitmq_cluster
> ```
>
> `rabbitmqctl stop` 会将Erlang 虚拟机关闭，`rabbitmqctl stop_app` 只关闭 RabbitMQ 服务

join_cluster 命令有一个可选的参数 --ram ，该参数代表新加入的节点是内存节点，默认是磁盘节点。如果是内存节点，则所有的队列、交换器、绑定关系、用户、访问权限和 vhost 的元数据都将存储在内存中，如果是磁盘节点，则存储在磁盘中。内存节点可以有更高的性能，但其重启后所有配置信息都会丢失，因此RabbitMQ 要求在集群中至少有一个磁盘节点，其他节点可以是内存节点。当内存节点离开集群时，它可以将变更通知到至少一个磁盘节点；然后在其重启时，再连接到磁盘节点上获取元数据信息。除非是将 RabbitMQ 用于 RPC 这种需要超低延迟的场景，否则在大多数情况下，RabbitMQ 的性能都是够用的，可以采用默认的磁盘节点的形式。这里为了演示， **`k8s-node2`我就设置为内存节点。**

另外，如果节点以磁盘节点的形式加入，则需要先使用 reset 命令进行重置，然后才能加入现有群集，重置节点会删除该节点上存在的所有的历史资源和数据。**采用内存节点的形式加入时可以略过 `reset `这一步，并在`join_cluster`时添加`--ram`参数，因为内存上的数据本身就不是持久化的。**

## 7. 查看集群状态

```
rabbitmqctl cluster_status
```

## 8. 配置镜像队列

### 1）开启镜像队列

> 为所有队列开启镜像配置

```
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

## 2）复制系数

> 在上面我们指定了 ha-mode 的值为 all ，代表消息会被同步到所有节点的相同队列中。这里我们之所以这样配置，因为我们本身只有三个节点，因此复制操作的性能开销比较小。如果你的集群有很多节点，那么此时复制的性能开销就比较大，此时需要选择合适的复制系数。通常可以遵循过半写原则，即对于一个节点数为 n 的集群，只需要同步到 n/2+1 个节点上即可。此时需要同时修改镜像策略为 exactly，并指定复制系数 ha-params

```
rabbitmqctl set_policy ha-two "^" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

除此之外，RabbitMQ 还支持使用正则表达式来过滤需要进行镜像操作的队列

```
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
```

## 9. 集群的关闭与重启

没有一个直接的命令可以关闭整个集群，需要逐一进行关闭。但是需要保证在重启时，最后关闭的节点最先被启动。如果第一个启动的不是最后关闭的节点，那么这个节点会等待最后关闭的那个节点启动，默认进行 10 次连接尝试，超时时间为 30 秒，如果依然没有等到，则该节点启动失败。

这带来的一个问题是，假设在一个三节点的集群当中，关闭的顺序为 node1，node2，node3，如果 node1 因为故障暂时没法恢复，此时 node2 和 node3 就无法启动。想要解决这个问题，可以先将 node1 节点进行剔除，命令如下：

```
rabbitmqctl forget_cluster_node rabbitmq1@k8s-node1 --offline
```

> `-offline` 参数，它允许节点在自身没有启动的情况下将其他节点剔除。
>
> 其中`rabbitmq1@k8s-node1`名称为可以通过查看进群方式查看
>
> ```
> rabbitmqctl cluster_status
> ```

## 10. 解除集群

重置当前节点

```
# 1.停止服务
rabbitmqctl stop_app
# 2.重置集群状态
rabbitmqctl reset
# 3.重启服务
rabbitmqctl start_app
```

重新加入集群

```
# 1.停止服务
rabbitmqctl stop_app
# 2.重置状态
rabbitmqctl reset
# 3.节点加入
rabbitmqctl join_cluster rabbitmq@k8s-node1
# 4.重启服务
rabbitmqctl start_app
```

完成后重新检查 RabbitMQ 集群状态

```
rabbitmqctl cluster_status
```

除了在当前节点重置集群外，还可在集群其他正常节点将节点踢出集群

```
rabbitmqctl forget_cluster_node rabbitmq@k8s-node1
```

## 11. 变更节点类型

我们可以将节点的类型从RAM更改为Disk，或者反之！

> 可以使用`change_cluster_node_type`命令。必须首先停止节点。

```
# 1.停止服务
rabbitmqctl stop_app
# 2.变更类型 ram disc
rabbitmqctl change_cluster_node_type disc
# 3.重启服务
rabbitmqctl start_app
```

## 12. 清除 RabbitMQ 节点配置

```
# 如果遇到不能正常退出直接kill进程
rabbitmqctl stop
# 查看进程
ps aux|grep rabbitmq
# 清楚节点rabbitmq配置
# rm -rf /var/lib/rabbitmq/mnesia
# 或者删除data下的数据文件
rm -rf /opt/rabbitmq/data/*
```
