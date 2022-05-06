---
layout: post
title: "RabbitMQ基础（三）- 环境部署"
description: "分享"
tag: RabbitMQ
---

# 1、单节点安装

官网地址 [https://www.rabbitmq.com/download.html](https://www.rabbitmq.com/download.html)

## 1. 安装rabbitmq依赖erlang环境

```
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

```
yum install -y socat
```

> Socat 是 Linux 下的一个多功能的网络工具，名字来由是 「Socket CAT」。其功能与有瑞士军刀之称的 Netcat 类似，可以看做是 Netcat 的加强版。

## 3. 解压rabbitmq并修改配置文件

```
tar xf rabbitmq-server-generic-unix-3.10.0.tar.xz -C /opt/
mv rabbitmq_server-3.10.0/ rabbitmq
```

### 1）添加配置文件：rabbitmq-env.conf

```
cat /opt/rabbitmq/etc/rabbitmq/rabbitmq-env.conf
RABBITMQ_NODENAME=rabbitmq001
RABBITMQ_NODE_IP_ADDRESS=192.168.6.21
RABBITMQ_NODE_PORT=5672
RABBITMQ_MNESIA_BASE=/opt/rabbitmq/data
RABBITMQ_LOG_BASE=/opt/rabbitmq/logs
```

创建目录：

```
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
cat /opt/rabbitmq/etc/rabbitmq/rabbitmq.config
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

### 3）配置用户并修改权限

```
useradd -u 1020 -s /sbin/nologin rabbitmq
chown -R rabbitmq:rabbitmq -R /opt/rabbitmq
```

### 4）使用普通用户启动测试

```
# 启动测试
su -s /bin/bash - rabbitmq
/opt/rabbitmq/sbin/rabbitmq-server start

# 停止使用stop
```

> 注意：出现completed with，表示启动成功

查看cookie文件是否存在

**注意：此文件必须存在**

```
ls -a /home/rabbitmq/.erlang.cookie
```

## 4. 配置环境变量

```
echo 'export PATH=/opt/rabbitmq/sbin:$PATH' >> /etc/profile.d/my_env.sh

source /etc/profle
```

## 5. 启动web管理界面

```
rabbitmq-plugins enable rabbitmq_management
```

### 1）创建账号并授权访问

> guest的默认账号受限，默认只能localhost 浏览器访问管理界面，否则最初是登不上去的。

```
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



