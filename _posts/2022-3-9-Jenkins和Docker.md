---
layout: post
title: "Jenkins基础（二）- Jenkins、Docker和SpringCloud持续集成"
description: "分享"
tag: Jenkins
---

# 1、 Jenkins+Docker+SpringCloud微服务持续集成（一）

## 1. Jenkins+Docker+SpringCloud持续集成流程说明

![image-20220308200219832](/images/posts/2022-3-3-Jenkins/image-20220308200219832.png)



**大致流程说明：**

> 1）开发人员每天把代码提交到Gitlab代码仓库 
>
> 2）Jenkins从Gitlab中拉取项目源码，编译并打成jar包，然后构建成Docker镜像，将镜像上传到 Harbor私有仓库。 
>
> 3）Jenkins发送SSH远程命令，让生产部署服务器到Harbor私有仓库拉取镜像到本地，然后创建容器。 
>
> 4）最后，用户可以访问到容器

**服务列表**

| 服务器名称       | IP           | 软件                      |
| ---------------- | ------------ | ------------------------- |
| 代码托管服务     | 192.168.6.61 | Gitlab                    |
| 持续集成服务     | 192.168.6.20 | Jenkins，Maven，Docker-ce |
| Docker仓库服务器 | 192.168.6.62 | Docker-ce、Harbor         |
| 生产部署服务器   | 192.168.6.63 | Docker-ce                 |

## 2. SpringCloud微服务源码概述（项目结构）

项目架构：前后端分离 

后端技术栈：SpringBoot+SpringCloud+SpringDataJpa（Spring全家桶） 

微服务项目结构：

| 项目名                  | 端口  | 说明                                        |
| ----------------------- | ----- | ------------------------------------------- |
| tensquare_parent        |       | 父工程                                      |
| tensquare_eureka_server | 10086 | 注册中心微服务                              |
| tensquare_zuul          | 10020 | 网关服务                                    |
| tensquare_admin_service | 9001  | 础权限认证中心，负责用户认证（使用JWT认证） |
| tensquare_gathering     | 9002  | 一个简单的业务模块，活动微服务相关逻辑      |
| tensquare_common        |       | 公共子工程，存放一些工具类等                |

![image-20220309114459177](/images/posts/2022-3-9-Jenkins和Docker/image-20220309114459177.png)

> tensquare_parent：父工程，存放基础配置 
>
> tensquare_common：通用工程，存放工具类 
>
> tensquare_eureka_server：SpringCloud的Eureka注册中心 （1）
>
> tensquare_zuul：SpringCloud的网关服务 	（2）
>
> tensquare_admin_service：基础权限认证中心，负责用户认证（使用JWT认证） （3）
>
> tensquare_gathering：一个简单的业务模块，活动微服务相关逻辑	（4）

- 测试启动顺序按照后面的数字

数据库结构：

> tensquare_user：用户认证数据库，存放用户账户数据。对应tensquare_admin_service微服务 
>
> ```
> # 建库
> create database tensquare_user;
> # 建表
> CREATE TABLE `tensquare_user`.`tb_admin` ( `id` CHAR(11), `loginname` CHAR(11), `password` CHAR(11), `state` CHAR(11) ); 
> 
> # 更新字段
> ALTER TABLE `tensquare_user`.`tb_admin` CHANGE `loginname` `loginname` VARCHAR(60) CHARSET utf8mb4 COLLATE utf8mb4_general_ci NULL, CHANGE `password` `password` VARCHAR(60) CHARSET utf8mb4 COLLATE utf8mb4_general_ci NULL; 
> # 插入数据
> INSERT INTO `tensquare_user`.`tb_admin` (`id`, `loginname`, `password`, `state`) VALUES ('14425', 'admin', '123456', '1');
> 
> # 更新使用bcrypt在线加密工具加密后的密码（具体请看错误3）
> UPDATE `tensquare_user`.`tb_admin` SET `password` = '$2a$10$6a7Grp1/8MFTjn1iVE.OjOQVCv01sp3I87a4u9JZ4G7NL09WNSgv.' WHERE `id` = '14425' AND `loginname` = 'admin' AND `password` = '123456' AND `state` = '1'; 
> ```
>
> 
>
> tensquare_gathering：活动微服务数据库。对应tensquare_gathering微服务
>
> ```
> # 创建数据库
> create database tensquare_gathering;
> 
> # 创建表  tb_gathering
> CREATE TABLE `tb_gathering`  (
> `id` varchar(11) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '编号',
> `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '活动名称',
> `summary` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL COMMENT '大会简介',
> `detail` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL COMMENT '详细说明',
> `sponsor` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '主办方',
> `starttime` date NULL DEFAULT NULL COMMENT '开始时间',
> `endtime` date NULL DEFAULT NULL COMMENT '截止时间',
> `address` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '举办地点',
> `enrolltime` datetime(6) NULL DEFAULT NULL COMMENT '报名截止',
> `state` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '是否可见',
> `city` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '城市',
> `image` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '活动图片',
> PRIMARY KEY (`id`) USING BTREE
> ) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
> 
> # 插入数据
> INSERT INTO `tb_gathering` VALUES ('1', 'aaa', NULL, NULL, 'sss', '2022-03-10', '2022-03-10', 'ccc', NULL, '1', '1', NULL);
> INSERT INTO `tb_gathering` VALUES ('2', 'geray', NULL, NULL, 'vvvv', '2022-03-10', '2022-03-10', 'cc', NULL, '1', '2', NULL);
> 
> 
> # 创建表  tb_city
> 
> ```



**测试时连接数据库服务错误处理**1：（[Caused by: javax.net.ssl.SSLHandshakeException: No appropriate protocol](https://www.cnblogs.com/musecho/p/15074718.html)）

> URL后面添加useSSL=false （url: jdbc:mysql://192.168.6.20:3306/tensquare_user?characterEncoding=UTF8&useSSL=false）

**测试时连接数据库服务错误处理2**（[No operations allowed after connection closed](https://cloud.tencent.com/developer/article/1707872) ）

> YML配置文件中添加如下：
>
> ```
> 
>   datasource:  
>     driverClassName: com.mysql.jdbc.Driver
>     url: jdbc:mysql://192.168.6.20:3306/tensquare_user?characterEncoding=UTF8&useSSL=false
>     username: root
>     password: root@123
>     hikari:
>       # 连接池最大连接数，默认是 10
>       maximum-pool-size: 60
>       # 链接超时时间，默认 30000(30 秒)
>       connection-timeout: 60000
>       # 空闲连接存活最大时间，默认 600000(10 分钟)
>       idle-timeout: 60000
>       # 连接将被测试活动的最大时间量
>       validation-timeout: 3000
>       # 此属性控制池中连接的最长生命周期，值 0 表示无限生命周期，默认 1800000(30 分钟)
>       max-lifetime: 60000
>       # 连接到数据库时等待的最长时间(秒)
>       login-timeout: 5
>       # 池中维护的最小空闲连接数
>       minimum-idle: 10
> ```

**测试时连接数据库服务错误处理3**（[Encoded password does not look like BCrypt](https://www.codejava.net/frameworks/spring/encoded-password-does-not-look-like-bcrypt)）

> BCrypt在线加密生成器：[https://www.jisuan.mobi/p163u3BN66Hm6JWx.html](https://www.jisuan.mobi/p163u3BN66Hm6JWx.html)

![image-20220309191531188](/images/posts/2022-3-9-Jenkins和Docker/image-20220309191531188.png)

加密后放入数据库中：

![image-20220309191608283](/images/posts/2022-3-9-Jenkins和Docker/image-20220309191608283.png)



微服务配置分析： 

> tensquare_eureka 
>
> tensquare_zuul 
>
> tensquare_admin_service 
>
> tensquare_gathering

启动微服务错误处理：

> JwtUtil.java的注解（ConfigurationProperties）报错，添加注解Component

## 3. 本地部署（1） - SpringCloud微服务部署

### 1）本地运行微服务

分别启动一下服务引导类：

> tensquare_eureka_server ---->   EurekaServerApplication.java  
>
> tensquare_zuul   ----->  ZuulApplication.java 
>
> tensquare_admin_service    ----->   AdminApplication.java
>
> tensquare_gathering  ----->  GatheringApplication.java

![image-20220309170338909](/images/posts/2022-3-9-Jenkins和Docker/image-20220309170338909.png)

### 2）使用postman测试功能是否可用

测试并获取token

![image-20220310090550015](/images/posts/2022-3-9-Jenkins和Docker/image-20220310090550015.png)

> 获取的token，后面访问每一个微服务时需要携带的认证信息：
>
> ```
> eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiIxNDQyNSIsInN1YiI6ImFkbWluIiwiaWF0IjoxNjQ2ODc0MzExLCJyb2xlcyI6ImFkbWluIiwiZXhwIjoxNjQ2ODc2MTExfQ.Qhr-BYHfg_0YqlwebXKi56KheUnvTdczqKU2sxA5-Ss
> ```

模拟前端进行访问：

> GET请求  ---->  Headers  （填入token和获取的认证信息）

![image-20220310105918729](/images/posts/2022-3-9-Jenkins和Docker/image-20220310105918729.png)



### 3）本地部署微服务

#### 0> SpringBoot项目修改内嵌Tomcat版本

> 打开pom.xml配置文件，解析父级依赖（查看spring-boot-starter-parent参数的版本） --- 我这里的版本2.0.1.RELEASE
>
> 进入仓库位置并找到依赖文件；例如我这里：repository\org\springframework\boot\spring-boot-dependencies\2.0.1.RELEASE  
>
> 找到pom后缀的文件，打开并搜素  tomcat.version
>
> 直接修改版本号（建议备份）

#### **1> SpringBoot微服务项目打包**

1. 引入SpringBoot插件

> 引入SpringBoot插件对SpringCloud项目打包（位于父工程的pom.xml文件中）

```
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

2. 对每个微服务工程进行打包

> 进入IDEA中，可以将需要打包的微服务拖入终端（进入子项目的目录）
>
> 执行打包命令：
>
> `mvn clean package`

- 打包后在target目录下产生相应的jar文件

> 这里的实例项目共一个注册中心和3个微服务

- 打包注册中心服务

![image-20220310152527223](/images/posts/2022-3-9-Jenkins和Docker/image-20220310152527223.png)

- 同样的方式打包tensquare_zuul 、tensquare_admin_service和tensquare_gathering

#### **2> 本地运行微服务的jar包**

> `java -jar xxx.jar` 

- 拷贝打包的jar包文件到工作目录（例如我这里随便选择一个目录：E:\workspace\springcloudTest）

> 命令窗口进入该路径执行java命令分别启动服务
>
> java -jar tensquare_eureka_server-1.0-SNAPSHOT.jar
>
> java -jar tensquare_zuul-1.0-SNAPSHOT.jar
>
> java -jar tensquare_admin_service-1.0-SNAPSHOT.jar
>
> java -jar tensquare_gathering-1.0-SNAPSHOT.jar



#### **3> 查看效果**

- 启动微服务

![image-20220310153800576](/images/posts/2022-3-9-Jenkins和Docker/image-20220310153800576.png)

- 查看效果

![image-20220310153841809](/images/posts/2022-3-9-Jenkins和Docker/image-20220310153841809.png)



## 4. 本地部署（2）- 前端静态web网站

前端技术栈：NodeJS + VueJS +  ElementUI

使用VSCode打开源码

1）本地运行

> npm run dev

2）打包静态web网站

> npm run build

打包后，产生dist目录的静态文件

3）部署到Nginx服务

把dist目录的静态文件拷贝到nginx的html目录，启动nginx

4）启动Nginx并访问

### 0）Node安装

官方网址：[https://nodejs.org/en/download/](https://nodejs.org/en/download/)

1. 下载.msi安装文件

![image-20220310164403804](/images/posts/2022-3-9-Jenkins和Docker/image-20220310164403804.png)



> **这里我使用的版本是：https://nodejs.org/download/release/v15.14.0/**
>
> 并创建两个目录：
>
> node_cache：缓存路径放
>
> node_global：全模块所在路径

![image-20220310200208603](/images/posts/2022-3-9-Jenkins和Docker/image-20220310200208603.png)

2. 设置全局目录和缓存目录

```
# 配置全局目录
npm config set prefix "D:\Program Files\nodejs\node_global"

# 配置缓存目录
npm config set cache "D:\Program Files\nodejs\node_cache"
```



3. 设置环境变量

> A：在【系统变量】下新建【NODE_PATH】；值：D:\Program Files\nodejs\node_modules
>
> B：将【用户变量】下的【Path】中的npm修改为：D:\Program Files\nodejs\node_global

![image-20220310201417711](/images/posts/2022-3-9-Jenkins和Docker/image-20220310201417711.png)

修改：

![image-20220310201356844](/images/posts/2022-3-9-Jenkins和Docker/image-20220310201356844.png)



3. 测试

> win + r 打开命令窗口
>
> ```
> npm -v
> node -v
> npm install express -g # -g是全局安装的意思
> ```





4. 设置仓库（使用淘宝仓库）；

```
npm config set registry http://registry.npm.taobao.org/
```



5. 安装node-sass

```
# 安装node-sass
cnpm install node-sass --save
```



### 1）前端配置修改

1. dev.env.js （网关地址为本地微服务zuul网关地址）

![image-20220310161516891](/images/posts/2022-3-9-Jenkins和Docker/image-20220310161516891.png)

2. prod.env.js（网关地址为后台服务网关地址）

![image-20220310161955944](/images/posts/2022-3-9-Jenkins和Docker/image-20220310161955944.png)

### 2）修改完成之后，本地运行并访问测试

> VSCode终端执行命令：
>
> npm run dev 

![image-20220310210439856](/images/posts/2022-3-9-Jenkins和Docker/image-20220310210439856.png)

访问：（密码和之前postman测试时使用的密码一样）

![image-20220310210455193](/images/posts/2022-3-9-Jenkins和Docker/image-20220310210455193.png)

### 3）打包静态web网站（部署到本地nginx时需要修改prod.env.js的网关）

> npm run build
>
> 打包后，产生dist目录的静态文件

![image-20220310211850900](/images/posts/2022-3-9-Jenkins和Docker/image-20220310211850900.png)

### 4）部署到nginx服务器

> 把dist目录的静态文件拷贝到nginx的html目录，
>
> 修改nginx的端口为82，并启动nginx

![image-20220310213158400](/images/posts/2022-3-9-Jenkins和Docker/image-20220310213158400.png)

### 5）启动nginx，并访问

> http://localhost:82

![image-20220310213150102](/images/posts/2022-3-9-Jenkins和Docker/image-20220310213150102.png)



## 5. 环境准备（1） - Docker部署

戳我哦这里：[Docker基础](https://geray-zsg.github.io/2022/01/Docker%E5%9F%BA%E7%A1%80/)

### 1）Docker部署

> 192.168.6.20、192.168.6.62、192.168.6.63

1. 卸载旧版本Docker

```
# 列出当前所有docker的包
yum list installed | grep docker

# 卸载docker包
yum -y remove <docker的包名称>

# 删除docker的所有镜像和容器
rm -rf /var/lib/docker 
```



2. 安装必要的软件包

```
sudo yum install -y yum-utils \
	device-mapper-persistent-data \
	lvm2
```



3. 设置下载的镜像仓库

```
sudo yum-config-manager \
	--add-repo \
	https://download.docker.com/linux/centos/docker-ce.repo
```

4. 列出需要安装的版本列表

```
yum list docker-ce --showduplicates | sort -r
```

5. 安装指定版本（这里使用最新版本）

```
sudo yum install docker-ce docker-ce-cli containerd.io -y

# 可以指定版本安装（比如：19.3.9）
sudo yum install docker-ce-19.03.9-3.el7 \
	docker-ce-cli-19.03.9-3.el7 \
    containerd.io -y
```

6. 查看版本

```
docker -v
```

7. 启动并设置开机自启

```
# 启动
sudo systemctl start docker 
# 设置开机启动
sudo systemctl enable docker
```

8. 添加阿里云镜像下载地址 

```
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://3fc19s4g.mirror.aliyuncs.com"]
}
EOF
```

9. 重启Docker

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 6. 环境准备（2） -  Dockerfile制作微服务镜像

> 上传Eureka微服务的jar包到linux
>
> ```
> mkdir -p images/{eureka,zuul,admin-service,gathering}
> tree /root/images
> ```

![image-20220311101826459](/images/posts/2022-3-9-Jenkins和Docker/image-20220311101826459.png)



### 1）Eureka注册中心镜像

1. 编写Dockerfile

   ```
   FROM openjdk:8-jdk-alpine
   ARG JAR_FILE
   COPY ${JAR_FILE} app.jar
   EXPOSE 10086
   ENTRYPOINT ["java","-jar","/app.jar"]
   ```

2. 镜像构建

   ```
   docker build --build-arg JAR_FILE=tensquare_eureka_server-1.0-SNAPSHOT.jar -t eureka:v1 .
   ```

3. 查看镜像是否创建成功

   ```
   docker images
   ```

   ![image-20220311102525066](/images/posts/2022-3-9-Jenkins和Docker/image-20220311102525066.png)

4. 创建容器

   ```
   docker run -i --name=eureka -p 10086:10086 eureka:v1
   ```

5. 访问容器

   > http://192.168.6.20:10086/



### 2）构建其他三个微服务镜像

```
docker build --build-arg JAR_FILE=tensquare_zuul-1.0-SNAPSHOT.jar -t zuul:v1 .

docker build --build-arg JAR_FILE=tensquare_admin_service-1.0-SNAPSHOT.jar -t admin-service:v1 .

docker build --build-arg JAR_FILE=tensquare_gathering-1.0-SNAPSHOT.jar -t gathering:v1 .
```



## 7. 环境准备（3） - Harbor镜像仓库搭建

### 1）Harbor部署

> 192.168.6.62节点

1. 安装docker-compose，并添加可执行权限

   ```
   sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   ```

2. 查看docker-compose是否安装成功

   ```
   docker-compose -version
   ```

3. 下载Harbor的压缩包（这里使用版本：v2.4.1）

   > [https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)

4. 上传压缩包到linux，并解压

   ```
   tar -xzf harbor-2.4.1.tar.gz
   mkdir /opt/harbor
   mv harbor/* /opt/harbor
   cd /opt/harbor
   cp harbor.yml.tmpl  harbor.yml
   ```

5. 修改Harbor的配置

   ```
   vi harbor.yml
   
   hostname: 192.168.6.62
   port: 85
   ```

   - 禁用https（或者配置https证书）

6. 安装

   ```
   ./prepare
   ./install.sh
   ```

7. 启动Harbor

   ```
   # 启动
   docker-compose up -d 
   docker-compose stop 
   docker-compose restart
   
   # 查看状态
   docker-compose ps
   ```

8. 访问Harbor

   > 默认账户密码：admin/Harbor12345
   >
   > 192.168.6.62:85

### 2）在Harbor创建用户和项目

#### 1. 创建项目

Harbor的项目分为公开和私有的： 

- 公开项目：所有用户都可以访问，通常存放公共的镜像，默认有一个library公开项目。 
- 私有项目：只有授权用户才可以访问，通常存放项目本身的镜像。 

> 给我们的微服务创建新的项目（私有）

![image-20220311144843186](/images/posts/2022-3-9-Jenkins和Docker/image-20220311144843186.png)

#### 2. 创建用户（geray/Geray@123）

![image-20220311145044883](/images/posts/2022-3-9-Jenkins和Docker/image-20220311145044883.png)

#### 3. 给私有项目分配用户

> 项目 ----> 成员  ----> +用户  ----> 新建用户名称和上面创建的保持一样 （开发人员）

| 角色       | 权限说明                                          |
| ---------- | ------------------------------------------------- |
| 项目管理员 | 除了读写权限，同时拥有用户管理/镜像扫描等管理权限 |
| 维护人员   | 对于指定项目拥有读写权限，创建 Webhooks           |
| 开发人员   | 对于指定项目拥有读写权限                          |
| 访客       | 对于指定项目拥有只读权限                          |

![image-20220311145246808](/images/posts/2022-3-9-Jenkins和Docker/image-20220311145246808.png)

#### 4. 以新用户登录Harbor

![image-20220311145606876](/images/posts/2022-3-9-Jenkins和Docker/image-20220311145606876.png)



### 3）把镜像上传到Harbor

#### 1. 镜像打标签

```
docker tag eureka:v1 192.168.6.62:85/tensquare/eureka:v1
```

#### 2. 推送镜像到Harbor仓库

```
docker push 192.168.6.62:85/tensquare/eureka:v1
```

> **错误信息：**
>
> The push refers to repository [192.168.6.62:85/tensquare/eureka]
> Get https://192.168.6.62:85/v2/: http: server gave HTTP response to HTTPS client

#### 3. 把Harbor地址加入到docker信任列表

```
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://3fc19s4g.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.6.62:85"]
}
```

> 重启Docker
>
> `systemctl restart docker`

#### 4. 再次执行推送命令，会提示权限不足（需要先登录Harbor仓库）

```
docker push 192.168.6.62:85/tensquare/eureka:v1
```

> 错误信息如下：
>
> The push refers to repository [192.168.6.62:85/tensquare/eureka]
> 694e3e08989c: Preparing
> ceaf9e1ebef5: Preparing
> 9b9b7f3d56a0: Preparing
> f1b5933fe4b5: Preparing
> unauthorized: unauthorized to access repository: tensquare/eureka, action: push: unauthorized to access repository: tensquare/eureka, action: push

#### 5. 登录Harbor重新推送即可

```
# 登录
docker login -u geray -p Geray@123 192.168.6.62:85
```

> 登录成功信息：
>
> WARNING! Using --password via the CLI is insecure. Use --password-stdin.
> WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
> Configure a credential helper to remove this warning. See
> https://docs.docker.com/engine/reference/commandline/login/#credentials-store
>
> Login Succeeded

- 重新推送即可

![image-20220311152540674](/images/posts/2022-3-9-Jenkins和Docker/image-20220311152540674.png)

### 4）从Harbor下载镜像

> 从192.168.6.63节点下载镜像

#### 1. 安装Docker，并启动Docker（已经完成）

#### 2. 修改Docker配置

> 添加镜像加速和Harbor仓库信任

```
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://3fc19s4g.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.6.62:85"]
}
```

> 重启服务
>
> `systemctl restart docker`

#### 3.  登录Harbor仓库，之后下载镜像

```
# 登录
docker login -u geray -p Geray@123 192.168.6.62:85

# 拉取
docker pull 192.168.6.62:85/tensquare/eureka:v1
```



## 8. 微服务持续集成（1）- 项目代码上传到Gitlab 

### 0）创建前后端项目

| 项目名          | 组          | 说明            |
| --------------- | ----------- | --------------- |
| tensquare_back  | geray_group | 后端微服务项目  |
| tensquare_front | geray_group | 前端web站点项目 |

#### 1. 创建组geray_group

![image-20220311160106844](/images/posts/2022-3-9-Jenkins和Docker/image-20220311160106844.png)

#### 2. 创建tensquare_back项目

> 在geray_group组中创建项目

![image-20220311160310254](/images/posts/2022-3-9-Jenkins和Docker/image-20220311160310254.png)

#### 3. 创建tensquare_front项目

> 在geray_group组中创建项目

![image-20220311160505224](/images/posts/2022-3-9-Jenkins和Docker/image-20220311160505224.png)

#### 4. 创建用户并添加到组成员（geray/Geray@123）

**注意：*添加用户后必须使用该用户登陆并重置密码测试后才可使用git的Remots配置***

> 创建用户：Admin ---> users --->   new user
>
> 添加到组：编辑组

- 创建用户

![image-20220311163627968](/images/posts/2022-3-9-Jenkins和Docker/image-20220311163627968.png)



- 添加到组

![image-20220311164011392](/images/posts/2022-3-9-Jenkins和Docker/image-20220311164011392.png)

### 1）后端微服务代码上传到Gitlab（tensquare_back）

1. 加入Git版本控制管理

> VCS  ---->  Enable Version Control Integration —> 选择Git（最终在项目的根目录下创建了一个本地仓库）

2. push到仓库

> 上一步选择Git，并下载完成之后继续
>
> 1、将本地代码添加代本地仓库暂存区：右击项目 —> 选择Git —> Add
>
> 2、将本地代码提交到本地仓库：右击项目 —> 选择Git —> Commit Directory （Commit Message，提交的描述信息）
>
> 3、添加远程仓库：右击项目 —> 选择Git —> Repository —> Remots  （tensquare_back /  http://192.168.6.61:82/geray_group/tensquare_back.git）
>
> 4、推送本地仓库代码到Gitlab：右击项目 —> 选择Git —> Repository —> Push —> 选择刚才添加的远程仓库地址即可

![image-20220311164802569](/images/posts/2022-3-9-Jenkins和Docker/image-20220311164802569.png)

### 2）前端web网站代码上传到Gitlab（tensquare_front）

```
# 项目目录下右击鼠标右键（Git Bash here）
# 1. 初始化项目
git init

# 2. 添加Gitlab的仓库地址（geray/Geray@123）
git remote add tensquare_front http://192.168.6.61:82/geray_group/tensquare_front.git

# 3. 提交项目到本地暂存区
git add * 

# 4. 提交项目到本地仓库
git commit -m "初始化项目"

# 5. 提交项目到Gitlab仓库
git push -u tensquare_front master
```

![image-20220311171403642](/images/posts/2022-3-9-Jenkins和Docker/image-20220311171403642.png)

![image-20220311171438125](/images/posts/2022-3-9-Jenkins和Docker/image-20220311171438125.png)



## 9. 微服务持续集成（2）- 从Gitlab拉取项目源码

### 1）Jenkins拉取tensquare_back后端微服务项目

#### 1. tensquare_back（流水线项目）

> 使用分支参数：This project is parameterized  ---->  String Parameter

![image-20220311174115864](/images/posts/2022-3-9-Jenkins和Docker/image-20220311174115864.png)

> 流水线 --->  Pipeline script from SCM  ---> Git

![image-20220311172841496](/images/posts/2022-3-9-Jenkins和Docker/image-20220311172841496.png)



#### 2. tensquare_back项目根目录创建Jenkinsfile

- 生成拉取项目代码

  > 流水线语法 —> 片段生成器 —> 示例步骤选择（checkout: Check out from version control）

![image-20220311173808540](/images/posts/2022-3-9-Jenkins和Docker/image-20220311173808540.png)

> 指定构建分支参数

![image-20220311174655572](/images/posts/2022-3-9-Jenkins和Docker/image-20220311174655572.png)

**完整的Jenkinsfile：**

```
node {
    stage('拉取代码') {
       checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '4cc970a0-e96f-4200-97d6-bebdc8258546', url: 'git@192.168.6.61:geray_group/tensquare_back.git']]])
    }
}
```

**修改之后：**

```
//定义git凭证ID
def git_auth = "4cc970a0-e96f-4200-97d6-bebdc8258546"

//定义git的url地址
def git_url = "git@192.168.6.61:geray_group/tensquare_back.git"

node {
    stage('拉取代码') {
       checkout([$class: 'GitSCM', branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
    }
}
```

> 重新提交到Gitlab后构建测试

![image-20220311175835878](/images/posts/2022-3-9-Jenkins和Docker/image-20220311175835878.png)

![image-20220311180119864](/images/posts/2022-3-9-Jenkins和Docker/image-20220311180119864.png)





## 10. 微服务持续集成（3）- 提交到SonarQube代码审查

### 1）给tensquare_back项目，添加选择性参数（Choice Parameter）

> Jenkins ---->  项目  ---->  配置 ----> This project is parameterized  ---->  Choice Parameter（选择性参数）

![image-20220311181343940](/images/posts/2022-3-9-Jenkins和Docker/image-20220311181343940.png)

效果：

![image-20220311181409394](/images/posts/2022-3-9-Jenkins和Docker/image-20220311181409394.png)

### 2）每个微服务项目的根目录下添加sonar-project.properties

> 这里添加主要的4个微服务
>
> - sonar.projectKey
> - sonar.projectName
>
> 这两个参数保持唯一（设置为每个项目名称）

```
# must be unique in a given SonarQube instance
sonar.projectKey=tensquare_zuul
# this is the name and version displayed in the SonarQube UI. Was mandatory
prior to SonarQube 6.1.
sonar.projectName=tensquare_zuul
sonar.projectVersion=1.0
# Path is relative to the sonar-project.properties file. Replace "\" by "/" on
Windows.
# This property is optional if sonar.modules is set.
sonar.sources=.
sonar.exclusions=**/test/**,**/target/**
sonar.java.binaries=.
sonar.java.source=1.8
sonar.java.target=1.8
# 代码中是否已有target目录，没有则需要注释
# sonar.java.libraries=**/target/classes/**
# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```

![image-20220311182148260](/images/posts/2022-3-9-Jenkins和Docker/image-20220311182148260.png)

### 3）修改Jenkinsfile构建脚本

> 添加代码审查

```
//定义git凭证ID
def git_auth = "4cc970a0-e96f-4200-97d6-bebdc8258546"

//定义git的url地址
def git_url = "git@192.168.6.61:geray_group/tensquare_back.git"

node {
    stage('拉取代码') {
       checkout([$class: 'GitSCM', branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
    }
    stage('代码审查') {
       //引用当前Jenkins的SonarQube Scanner工具（Global Tool Configuration）
       def scannerHome = tool 'sonar-scanner'
       //引用当前Jenkins的SonarQube Server的环境（Configure System）
       withSonarQubeEnv('sonarqueb7.6') {
            //一下是shell脚本（${project_name}是Jenkins项目定义的选择性变量）
            //1.进入需要审查的项目根目录
            //2.调用sonar-scanner进行项目检测
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
       }
    }
}
```

- 提交到Gitlab并启动sonarqube；构建测试

```
cd /opt/sonar
su sonar ./bin/linux-x86-64/sonar.sh start

# 查看
su sonar ./bin/linux-x86-64/sonar.sh status

tail -f logs/sonar.log
```

![image-20220311184105641](/images/posts/2022-3-9-Jenkins和Docker/image-20220311184105641.png)



## 11. 微服务持续集成（4）- 使用Dockerfile编译、生成镜像

利用dockerfile-maven-plugin插件构建Docker镜像

### 1）编译安装公共子工程（tensquare_common）

1. Jenkinsfile文件添加执行步骤

```
    stage('编译、安装公共子工程') {
        sh 'mvn -f tensquare_common clean install'
    }
```

![image-20220314093745771](/images/posts/2022-3-9-Jenkins和Docker/image-20220314093745771.png)

推送到Gitlab并构建测试

![image-20220314114027882](/images/posts/2022-3-9-Jenkins和Docker/image-20220314114027882.png)

#### 构建失败的原因

> 项目的父工程中引用了spring-boot-maven-plugin插件；
>
> 该插件会对项目进行打包，tensquare_common作为tensquare_parent的子工程，也会进行springboot的打包；
>
> 但是tensquare_common并不是一个微服务工程，没有相关的启动类，所以在构建过程中，并不需要被打包（只有具备启动类的微服务工程才需要被打包）

![image-20220314114924015](/images/posts/2022-3-9-Jenkins和Docker/image-20220314114924015.png)

```
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```



#### 解决办法

> 将父工程pom.xml中的spring-boot-maven-plugin插件移动到每一个**需要被打包的微服务（具有启动类的）**工程中即可
>
> 构建成功

![image-20220314115840163](/images/posts/2022-3-9-Jenkins和Docker/image-20220314115840163.png)



### 2）编译、打包微服务工程

1. Jenkinsfile添加可执行步骤

```
    stage('编译、安装公共子工程') {
        sh 'mvn -f tensquare_common clean install'
    }
    stage('编译、打包微服务工程') {
        // ${project_name} 就是Jenkins构建时传入的微服务项目名称参数
        sh 'mvn -f ${project_name} clean package'
    }
```

![image-20220314120612999](/images/posts/2022-3-9-Jenkins和Docker/image-20220314120612999.png)

2. 构建测试（整个过程Ok），并在Jenkins的工作目录下生成了相关jar文件即可

![image-20220314120857047](/images/posts/2022-3-9-Jenkins和Docker/image-20220314120857047.png)

![image-20220314121052868](/images/posts/2022-3-9-Jenkins和Docker/image-20220314121052868.png)



#### 构建tensquare_zuul工程（网关）时错误：依赖的父工程找不到

> ```
> [ERROR] Failed to execute goal on project tensquare_zuul: Could not resolve dependencies for project com.tensquare:tensquare_zuul:jar:1.0-SNAPSHOT: Failed to collect dependencies at com.tensquare:tensquare_common:jar:1.0-SNAPSHOT: Failed to read artifact descriptor for com.tensquare:tensquare_common:jar:1.0-SNAPSHOT: Could not find artifact com.tensquare:tensquare_parent:pom:1.0-SNAPSHOT -> [Help 1]
> ```

![image-20220314121811821](/images/posts/2022-3-9-Jenkins和Docker/image-20220314121811821.png)

> 这是由于tensquare_zuul依赖tensquare_parent父工程，但是6.20机器上的maven仓库中并没有我们项目的父工程导致依赖失败

![image-20220314122054997](/images/posts/2022-3-9-Jenkins和Docker/image-20220314122054997.png)



#### 解决办法：手动将父工程的目录结构传到6.20上的maven仓库中

> 找到本地仓库中的父工程目录结构
>
> 根据本地的路径结构上传到6.20上的maven仓库中即可

本地的目录结构：

![image-20220314130219101](/images/posts/2022-3-9-Jenkins和Docker/image-20220314130219101.png)

> 可以看到该父工程的目录结构位于maven仓库中的`com/tensquare`目录下
>
> tensquare_parent整个目录，拷贝到6.20的maven仓库的`com/tensquare`中即可

![image-20220314130613869](/images/posts/2022-3-9-Jenkins和Docker/image-20220314130613869.png)

构建测试：

![image-20220314130810844](/images/posts/2022-3-9-Jenkins和Docker/image-20220314130810844.png)

构建ok



### 3）在每个微服务项目的pom.xml加入dockerfile-maven-plugin插件

利用dockerfile-maven-plugin插件构建Docker镜像

> 通过dockerfile-maven-plugin插件来去读取项目中的Dockerfile文件，并进行项目的镜像构建
>
> JAR_FILE：将内容作为参数传递给docker构建镜像的命令中；如：`docker build --build-arg JAR_FILE=tensquare_eureka_server-1.0-SNAPSHOT.jar -t eureka:v1 .`
>
> ${project.build.finalName}.jar  获取当前项目打包后的jar文件名称作为参数传递

```
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>1.3.6</version>
                <configuration>
                    <repository>${project.artifactId}</repository>
                    <buildArgs>
                        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
```



![image-20220314133823812](/images/posts/2022-3-9-Jenkins和Docker/image-20220314133823812.png)

### 4）在每个微服务项目根目录下建立Dockerfile文件

```
#FROM java:8
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
EXPOSE 10086
ENTRYPOINT ["java","-jar","/app.jar"]
```

- 注意：每个微服务的端口等信息唯一

![image-20220314133929070](/images/posts/2022-3-9-Jenkins和Docker/image-20220314133929070.png)

### 5）修改Jenkinsfile构建脚本

```
    stage('编译、安装公共子工程') {
        sh 'mvn -f tensquare_common clean install'
    }
    stage('编译、打包微服务工程、构建docker镜像') {
        // ${project_name} 就是Jenkins构建时传入的微服务项目名称参数
        // 添加构建镜像的操作
        sh 'mvn -f ${project_name} clean package dockerfile:build'
    }
```

![image-20220314135748845](/images/posts/2022-3-9-Jenkins和Docker/image-20220314135748845.png)



### 6）构建测试

1. 清理镜像

![image-20220314135520909](/images/posts/2022-3-9-Jenkins和Docker/image-20220314135520909.png)



2. 构建测试

> 可以看到Jenkins的一些构建信息

![image-20220314140154024](/images/posts/2022-3-9-Jenkins和Docker/image-20220314140154024.png)

3. 查看镜像，已成功构建

![image-20220314140233541](/images/posts/2022-3-9-Jenkins和Docker/image-20220314140233541.png)



4. 启动镜像并访问测试

```
docker run -i --name=eureka -p 10086:10086 tensquare_eureka_server:latest
```

![image-20220314140428360](/images/posts/2022-3-9-Jenkins和Docker/image-20220314140428360.png)



## 12. 微服务持续集成（5）- 上传镜像到Harbor仓库

### 1）使用凭证管理Harbor私服账户和密码

> manage jenkins  --->   Manage Credentials  --->  全局凭据  --->  添加凭据
>
> geray/Geray@123 ：Harbor中的用户名密码
>
> 最后保存生成的ID：4fc32a5b-bd23-4984-94c4-9b30f50fbfe5

![image-20220314153509698](/images/posts/2022-3-9-Jenkins和Docker/image-20220314153509698.png)

查看生成的ID，并保存（后面需要添加到Jenkinsfile中的登陆步骤）

![image-20220314153903767](/images/posts/2022-3-9-Jenkins和Docker/image-20220314153903767.png)



### 2）脚本生成器生成登陆Harbor的脚本信息

> 流水线语法  ---->   片段生成器  ----->   withCredentials Bind Credentials  to variables
>
> 自定义用户名和密码的变量名（分别对应凭据中保存的用户名和密码信息）以及选择Harbor仓库的凭据进行绑定，方便在Jenkinsfile中获取相关信息

![image-20220314155137357](/images/posts/2022-3-9-Jenkins和Docker/image-20220314155137357.png)

> 生成的脚本中的ID信息可以在Jenkinsfile中定义变量来引用，具体如下：

```
        // 推送镜像到Harbor仓库
        // ${harbor_auth} 引用harbor的凭证ID
       withCredentials([usernamePassword(credentialsId: '${harbor_auth}', passwordVariable: 'password', usernameVariable: 'username')]) {
            // some block
        }
```



### 3）修改Jenkinsfile构建脚本

> 1. 定义版本号、镜像名称、Harbor仓库地址、镜像仓库名称、Harbor登陆凭证
>
> 2. 镜像打标签 ：`docker tag 原镜像 目标镜像`（目标镜像：仓库地址/项目仓库名称/镜像名称:标签）
> 3. 登陆harbor仓库：`docker login -u ${username} -p ${password} ${harbor_url} `  （需要将Harbor的用户信息保存到Jenkins凭证）
> 4. 上传镜像：`docker push 镜像`
> 5. 删除本地镜像：`docker rmi 镜像id`

**完整的Jenkinsfile如下：**

```
//定义git凭证ID
def git_auth = "4cc970a0-e96f-4200-97d6-bebdc8258546"

//定义git的url地址
def git_url = "git@192.168.6.61:geray_group/tensquare_back.git"

// 定义镜像版本号
def tag = "latest"

// 定义Harbor的URL地址
def harbor_url = "192.168.6.62:85"

// 定义镜像仓库名称
def harbor_project_name = "tensquare"

// 定义Harbor的登陆凭证ID
def harbor_auth = "4fc32a5b-bd23-4984-94c4-9b30f50fbfe5"

node {
    stage('拉取代码') {
       checkout([$class: 'GitSCM', branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
    }
    stage('代码审查') {
       //引用当前Jenkins的SonarQube Scanner工具（Global Tool Configuration）
       def scannerHome = tool 'sonar-scanner'
       //引用当前Jenkins的SonarQube Server的环境（Configure System）
       withSonarQubeEnv('sonarqueb7.6') {
            //一下是shell脚本（${project_name}是Jenkins项目定义的选择性变量）
            //1.进入需要审查的项目根目录
            //2.调用sonar-scanner进行项目检测
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
       }
    }
    stage('编译、安装公共子工程') {
        sh 'mvn -f tensquare_common clean install'
    }
    stage('编译、打包微服务工程、构建docker镜像、上传镜像到Harbor仓库') {
        // ${project_name} 就是Jenkins构建时传入的微服务项目名称参数
        // 添加构建镜像的操作
        sh 'mvn -f ${project_name} clean package dockerfile:build'

        // 上传镜像到Harbor仓库
        // 定义镜像名称（项目名称+版本号）
        def imageName = "${project_name}:${tag}"

        // 打标签
        sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"

        // 推送镜像到Harbor仓库
        // ${harbor_auth} 引用harbor的凭证ID
        withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
             // 登陆到Harbor仓库
             sh "docker login -u ${username} -p ${password} ${harbor_url}"

             // 镜像上传
             sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"

            sh "echo 镜像上传成功"
        }

    }
}
```



构建测试：成功被构建并上传镜像到Harbor仓库

![image-20220314164122441](/images/posts/2022-3-9-Jenkins和Docker/image-20220314164122441.png)

上传成功：

![image-20220314164229633](/images/posts/2022-3-9-Jenkins和Docker/image-20220314164229633.png)



## 13. 微服务持续集成（6）- 镜像拉取和应用发布

- 确保需要用到的服务已成功部署docker并启动

### 1）Jenkins安装 Publish Over SSH 插

> 安装 Publish Over SSH插件，可以实现远程发送Shell命令
>
> 该插件同样会在流水线语法代码生成器中产生相关提示（sshPublisher: Send build artifacts over SSH）

![image-20220314165002306](/images/posts/2022-3-9-Jenkins和Docker/image-20220314165002306.png)

### 2）配置远程部署服务器

1. 拷贝公钥到远程服务器（生产部署服务器）

> 确保Jenkins服务和远程部署服务器之间能够实现远程调用，即实现免密登陆
>
> SSH远程调用需要公钥和私钥（6.20之前已经产生过公钥和私钥）,命令如下：
>
> ```
> # 一路回车即可
> ssh-keygen -t rsa
> ```
>
> **在/root/.ssh/目录保存了公钥和使用**
>
> id_rsa：私钥文件
>
> id_rsa.pub：公钥文件

- 也可以直接将公钥文件内容复制值远程服务器对应的文件中（.ssh/authorized_keys）

```
# Jenkins服务上的公钥推送到远程部署服务器上
ssh-copy-id 192.168.6.63
```

2. 系统配置  ->  添加远程服务器

> Manage Jenkins --->   Configure System  --->  Publish over SSH



![image-20220314172131046](/images/posts/2022-3-9-Jenkins和Docker/image-20220314172131046.png)



### 3）Jenkins项目定义port变量

> 每个微服务的端口都是唯一的，根据不同的微服务传入对应port端口，在Jenkinsfile中动态的根据变量获取并执行deploy.sh脚本进行部署

> 项目名 --->  配置  ---->  This project is parameterized  ----->  添加参数  ----->  string Parameter

![image-20220314175346725](/images/posts/2022-3-9-Jenkins和Docker/image-20220314175346725.png)



### 4）编写deploy.sh部署脚本，并上传到部署服务器

> 用于Jenkinsfile中构建步骤过程中触发该脚本来执行一些部署动作
>
> 该脚本需要接收7个参数（harbor_url、harbor_project_name、project_name、tag、port、username和password）
>
> - 其中project_name 和 port是Jenkins构建时传入的每个微服务的名称和端口号！
>
> - username和password是Harbor仓库登陆时所需的用户信息

```
#! /bin/sh
#接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4
port=$5
username=$6
password=$7

imageName=$harbor_url/$harbor_project_name/$project_name:$tag

echo "$imageName"

#查询容器是否存在，存在则删除
containerId=`docker ps -a | grep -w ${project_name}:${tag} | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
	#停掉容器
	docker stop $containerId
	
	#删除容器
	docker rm $containerId
	
	echo "成功删除容器"
fi

#查询镜像是否存在，存在则删除
imageId=`docker images | grep -w $project_name | awk '{print $3}'`

if [ "$imageId" != "" ] ; then
	#删除镜像
	docker rmi -f $imageId
	
	echo "成功删除镜像"
fi

# 登录Harbor私服
# docker login -u geray -p Geray@123 $harbor_url
docker login -u ${username} -p ${password} ${harbor_url}

# 下载镜像
docker pull $imageName

# 启动容器
docker run -di -p $port:$port $imageName

echo "容器启动成功"
```

- 上传deploy.sh文件到/opt/jenkins_shell目录下，且文件至少有执行权限！

```
mkdir /opt/jenkins_shell

# 添加执行权限
chmod +x /opt/jenkins_shell/deploy.sh 
```



### 5）流水线语法生成部署所需的脚本代码

> 流水线  ---->  流水线语法   ----->  代码生成器  ----> sshPublisher: Send build artifacts over SSH
>
> 添加命令：`/opt/jenkins_shell/deploy.sh $harbor_url $harbor_project_name $project_name $tag $port $username $password`

![image-20220314182733523](/images/posts/2022-3-9-Jenkins和Docker/image-20220314182733523.png)

生成的脚本如下：（执行命令建议使用双引号，存在变量）

![image-20220314182929872](/images/posts/2022-3-9-Jenkins和Docker/image-20220314182929872.png)

- execCommand修改为双引号

```
sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/opt/jenkins_shell/deploy.sh $harbor_url $harbor_project_name $project_name $tag $port $username $password", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
```



### 6）修改Jenkinsfile构建脚本

- 完整的Jenkinsfile如下：

```
//定义git凭证ID
def git_auth = "4cc970a0-e96f-4200-97d6-bebdc8258546"

//定义git的url地址
def git_url = "git@192.168.6.61:geray_group/tensquare_back.git"

// 定义镜像版本号
def tag = "latest"

// 定义Harbor的URL地址
def harbor_url = "192.168.6.62:85"

// 定义镜像仓库名称
def harbor_project_name = "tensquare"

// 定义Harbor的登陆凭证ID
def harbor_auth = "4fc32a5b-bd23-4984-94c4-9b30f50fbfe5"

node {
    stage('拉取代码') {
       checkout([$class: 'GitSCM', branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
    }
    stage('代码审查') {
       //引用当前Jenkins的SonarQube Scanner工具（Global Tool Configuration）
       def scannerHome = tool 'sonar-scanner'
       //引用当前Jenkins的SonarQube Server的环境（Configure System）
       withSonarQubeEnv('sonarqueb7.6') {
            //一下是shell脚本（${project_name}是Jenkins项目定义的选择性变量）
            //1.进入需要审查的项目根目录
            //2.调用sonar-scanner进行项目检测
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
       }
    }
    stage('编译、安装公共子工程') {
        sh 'mvn -f tensquare_common clean install'
    }
    stage('编译、打包微服务工程、构建docker镜像、上传镜像到Harbor仓库') {
        // ${project_name} 就是Jenkins构建时传入的微服务项目名称参数
        // 添加构建镜像的操作
        sh 'mvn -f ${project_name} clean package dockerfile:build'

        // 上传镜像到Harbor仓库
        // 定义镜像名称（项目名称+版本号）
        def imageName = "${project_name}:${tag}"

        // 打标签
        sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"

        // 推送镜像到Harbor仓库
        // ${harbor_auth} 引用harbor的凭证ID
        withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
             // 登陆到Harbor仓库
             sh "docker login -u ${username} -p ${password} ${harbor_url}"

             // 镜像上传
             sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"

            sh "echo 镜像上传成功"
        }
        //删除本地镜像
        sh "docker rmi -f ${imageName}"
        sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${imageName}"

        //=====以下为远程调用进行项目部署========
        sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/opt/jenkins_shell/deploy.sh $harbor_url $harbor_project_name $project_name $tag $port", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])


    }
}
```

修改相关微服务中的eureka服务配置信息，并构建测试



### 7）修改每个微服务项目的Eureka服务配置信息（application.yml）

> 将本地localhost或者127.0.0.1修改远程服务对外暴露的eureka信息

zuul：

![image-20220314212029499](/images/posts/2022-3-9-Jenkins和Docker/image-20220314212029499.png)

admin_service：

![image-20220314212143533](/images/posts/2022-3-9-Jenkins和Docker/image-20220314212143533.png)

tensquare_gathering：

![image-20220314212748515](/images/posts/2022-3-9-Jenkins和Docker/image-20220314212748515.png)

**最终启动结果：**

![image-20220314213141176](/images/posts/2022-3-9-Jenkins和Docker/image-20220314213141176.png)



### 8）使用postman测试

1. POST请求获取token

![image-20220314214129098](/images/posts/2022-3-9-Jenkins和Docker/image-20220314214129098.png)

2. GET请求测试，获取数据

![image-20220314214329131](/images/posts/2022-3-9-Jenkins和Docker/image-20220314214329131.png)



## 14. 微服务持续集成（7）- 部署前端静态web网站

![image-20220314215052363](/images/posts/2022-3-9-Jenkins和Docker/image-20220314215052363.png)

### 1）部署Nginx服务器（远程部署服务器）

```
# 安装依赖
yum install epel-release -y
# 安装
yum -y install nginx 
```

- 修改nginx的端口，默认80，改为9090：

```
vi /etc/nginx/nginx.conf
    server {
        listen       9090;
        listen       [::]:9090;
        server_name  _;
        root         /usr/share/nginx/html;
```

- 还需要关闭selinux，将SELINUX=disabled

```
# 先临时关闭
setenforce 0 
# 编辑文件，永久关闭 SELINUX=disabled
vi /etc/selinux/config 
```

- 启动Nginx

```
# 设置开机启动
systemctl enable nginx 
# 启动
systemctl start nginx 
# 停止
systemctl stop nginx 
# 重启
systemctl restart nginx
```

访问：http://192.168.6.63:9090/





### 2）安装NodeJS插件，并配置NodeJS的环境

> 前台web站点是NodeJS开发的，所以Jenkins需要NodeJS的相关插件环境

![image-20220315093257680](/images/posts/2022-3-9-Jenkins和Docker/image-20220315093257680.png)

#### Jenkins服务器上部署NodeJS环境（编译安装）

> 选择相关版本下载：
>
> 这里使用：https://nodejs.org/download/release/v15.14.0/

- 安装依赖

> `gcc -v`：确保gcc版本在6版本以上
>
> gcc下载地址http://ftp.gnu.org/gnu/gcc/，找到需要版本的tar.gz包
>
> 这里使用7.1版本进行安装

```
# 安装依赖
yum -y install bzip2
yum install m4 -y
yum install gmp-devel.x86_64 -y
yum install mpfr-devel.x86_64 -y
yum install gcc-c++.x86_64 -y

# 下载并安装gcc
wget http://ftp.gnu.org/gnu/gcc/gcc-7.1.0/gcc-7.1.0.tar.gz
tar xf gcc-7.1.0.tar.gz -C /usr/local
 
cd /usr/local/gcc-7.1.0

# 执行下载依赖包
./contrib/download_prerequisites
# 特别慢，可以查看进程
ps -ef |grep wget

```

> gmp-6.1.0.tar.bz2: 确定
> mpfr-3.1.4.tar.bz2: 确定
> mpc-1.0.3.tar.gz: 确定
> isl-0.16.1.tar.bz2: 确定
> tar (child): lbzip2：无法 exec: 没有那个文件或目录
> tar (child): Error is not recoverable: exiting now
> tar: Child returned status 2
> tar: Error is not recoverable: exiting now
> error: Cannot extract package from gmp-6.1.0.tar.bz2
> [root@k8s-master1 gcc-7.1.0]# rm -rf gmp-6.1.0.tar.bz2

编译安装：

```
./configure --enable-checking=release --enable-languages=c,c++ --disable-multilib
# 这里执行需要时间很长，可能要几个小时去加载
make -j4
make install
```

> 如果显示的gcc版本仍是以前的版本，就需要重启系统

![image-20220315143742767](/images/posts/2022-3-9-Jenkins和Docker/image-20220315143742767.png)

> 更新gcc

```
sudo yum install centos-release-scl -y
sudo yum install devtoolset-7 -y
scl enable devtoolset-7 bash
```

![image-20220315151356990](/images/posts/2022-3-9-Jenkins和Docker/image-20220315151356990.png)

- 安装nodejs

  > 先安装Python3.6：https://www.cnblogs.com/hszstudypy/p/11510865.html

```
# 安装依赖
yum install freeglut -y
yum install freeglut-devel -y

# 下载并解压NodeJS
wget https://npm.taobao.org/mirrors/node/v15.14.0/node-v15.14.0.tar.gz
tar xf node-v15.14.0.tar.gz -C /usr/local

# 进入目录进行编译
cd /usr/local/node-v15.14.0

mkdir /usr/local/nodejs

# 编译安装 
./configure --prefix=/usr/local/nodejs
# -j4可以同时运行4个编译，可以帮助减少编译时间
make -j8
make install

# 配置全局和缓存目录
mkdir -p /usr/local/nodejs/{node_cache,node_global,node_modules}

# 设置环境变量
vi /etc/profile
# 追加一下内容
export NODE_HOME=/usr/local/nodejs
export PATH=$PATH:$NODE_HOME/bin:/usr/local/nodejs/node_global/bin/

# 加载配置
source /etc/profile

# 配置全局目录
npm config set prefix "/usr/local/nodejs/node_global"

# 配置缓存目录
npm config set cache "/usr/local/nodejs/node_cache"

# 查看版本
node -v

# 设置国内源下载
npm config set registry https://registry.npm.taobao.org

# 查看是否正常
npm config get registry 


```

**编译安装过程中的问题处理：**

参考链接：https://www.jianshu.com/p/b3b6eb9dc11a

> /usr/local/node-v15.14.0/out/Release/icupkg: /lib64/libstdc++.so.6: version `CXXABI_1.3.9' not found (required by /usr/local/node-v15.14.0/out/Release/icupkg)
> make[1]: *** [/usr/local/node-v15.14.0/out/Release/obj/gen/icudt68l.dat] 错误 1
> make[1]: 离开目录“/usr/local/node-v15.14.0/out”
> make: *** [node] 错误 2
> [root@k8s-master1 node-v15.14.0]# echo $?
> 2
> [root@k8s-master1 node-v15.14.0]# strings /usr/lib64/libstdc++.so.6 | grep GLIBC

![image-20220315144959365](/images/posts/2022-3-9-Jenkins和Docker/image-20220315144959365.png)

```
find /usr/local/gcc-7.1.0/stage1-x86_64-pc-linux-gnu/ -name "libstdc++.so*"

cp /usr/local/gcc-7.1.0/stage1-x86_64-pc-linux-gnu/libstdc++-v3/src/.libs/libstdc++.so.6.0.23 /usr/lib64

#切换工作目录至/usr/lib64，删除原来的软连接， 将默认库的软连接指向最新动态库。
cd /usr/lib64

[root@k8s-master1 lib64]# ll libstdc++.so.6
lrwxrwxrwx. 1 root root 19 2月  25 03:25 libstdc++.so.6 -> libstdc++.so.6.0.19

rm -rf libstdc++.so.6
ln -s libstdc++.so.6.0.23 libstdc++.so.6
```



> 安装相关插件

```
npm install express -g # -g是全局安装的意思

npm install -g cnpm --registry=https://registry.npm.taobao.org

# 安装node-sass
cnpm install node-sass --save

# npm run dev
```





#### 配置NodeJS环境

> Manage Jenkins ---->  Global Tool Configuration --->  NodeJS   ---->  新增NodeJS
>
> 根据上面的版本选择对应的版本
>
> 设置别名并选择版本即可

![image-20220315113818103](/images/posts/2022-3-9-Jenkins和Docker/image-20220315113818103.png)



### 3）创建前端流水线项目

#### 设置分支参数

![image-20220315094215186](/images/posts/2022-3-9-Jenkins和Docker/image-20220315094215186.png)

#### 添加流水线脚本（Jenkinsfile这里直接写入Jenkins项目环境中）

> 流水线脚本生成看下面：**建立Jenkinsfile构建脚本方式**

![image-20220315100655422](/images/posts/2022-3-9-Jenkins和Docker/image-20220315100655422.png)



### 4）建立Jenkinsfile构建脚本方式

#### 代码拉取

![image-20220315095746737](/images/posts/2022-3-9-Jenkins和Docker/image-20220315095746737.png)

生成的脚本：

```
checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '4fc32a5b-bd23-4984-94c4-9b30f50fbfe5', url: 'git@192.168.6.61:geray_group/tensquare_front.git']]])
```

![image-20220315095848662](/images/posts/2022-3-9-Jenkins和Docker/image-20220315095848662.png)

#### NodeJS打包构建脚本生成

![image-20220315114244896](/images/posts/2022-3-9-Jenkins和Docker/image-20220315114244896.png)

- 最后写入相关执行构建命令命令



#### 远程部署（目录拷贝）

> 之前的后端代码使用是执行脚本命令方式，这里使用目录拷贝方式
>
> 设置源码路径、远程发布目录

- 远程发布目录可以在nginx的配置文件找到（http.server.root）

![image-20220315095353219](/images/posts/2022-3-9-Jenkins和Docker/image-20220315095353219.png)

生成结果：

```
sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/usr/share/nginx/html', remoteDirectorySDF: false, removePrefix: '', sourceFiles: 'dist/**')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
```

![image-20220315095416948](/images/posts/2022-3-9-Jenkins和Docker/image-20220315095416948.png)

#### 最后合并在Jenkinsfile中

> 拉取代码中的用户凭据，定义为变量存放

```
//gitlab的凭证
def git_auth = "4fc32a5b-bd23-4984-94c4-9b30f50fbfe5"

node {
	stage('拉取代码') {
		checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: 'git@192.168.6.61:geray_group/tensquare_front.git']]])
	}
	stage('打包，构建网站') {
		//使用NodeJS的npm进行打包
		//引用Jenkins中配置的NodeJS环境别名
		nodejs('nodejs12'){
			//安装相关插件并打包构建
			sh '''
			npm install
			npm run build
			'''
		}
	}
	//=====以下为远程调用进行项目部署========
	sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/usr/share/nginx/html', remoteDirectorySDF: false, removePrefix: '', sourceFiles: 'dist/**')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
}
```



### 5）修改静态web网站的网关地址并构建测试

> 项目  ---->  config --->  prod.env.js
>
> - 项目根目录右键打开`Git Bash here`

```
#  1. 查看修改文件状态
git status -s

# 2. 将修改后的文件add到本地暂存区
git add config/prod.env.js

# 查看修改文件状态
git status 

# 3. 提交到本地库
git commint -m "修改远程服务网关地址"

# 查看修改文件状态
git status 

# 4. 提交到远程代码仓库
# 查看远程仓库地址
git remote -v

# 推送
git push -u tensquare_front master
```

![image-20220315103649177](/images/posts/2022-3-9-Jenkins和Docker/image-20220315103649177.png)



![image-20220315103725516](/images/posts/2022-3-9-Jenkins和Docker/image-20220315103725516.png)

#### Jenkins构建测试并访问



完成后，访问：http://192.168.6.63:9090 进行测试。 





# 2、 Jenkins+Docker+SpringCloud微服务持续集成（二）

## 1. Jenkins+Docker+SpringCloud部署方案优化

**上面部署方案存在的问题：** 

1）一次只能选择一个微服务部署 

2）只有一台生产者部署服务器 

3）每个微服务只有一个实例，容错率低

**优化方案：** 

1）在一个Jenkins工程中可以选择多个微服务同时发布

2）在一个Jenkins工程中可以选择多台生产服务器同时部署 

3）每个微服务都是以集群高可用形式部署



## 2. Jenkins+Docker+SpringCloud集群部署流程说明

![image-20220316092909530](/images/posts/2022-3-9-Jenkins和Docker/image-20220316092909530.png)

## 3. 修改所有微服务配置

### 1）eureka-注册中心配置(*)

>

```
# 集群版
spring:
  application:
    name: EUREKA-HA # 集群名称

---
server:
  port: 10086
spring:
  # 指定profile=eureka-server1
  profiles: eureka-server1
eureka:
  instance:
    # 指定当profile=eureka-server1时，主机名是eureka-server1
    hostname: 192.168.6.62
  client:
    service-url:
     # 将自己注册到eureka-server1、eureka-server2这个Eureka上面去
      defaultZone: http://192.168.6.62:10086/eureka/,http://192.168.6.63:10086/eureka/

---
server:
  port: 10086
spring:
  profiles: eureka-server2
eureka:
  instance:
    hostname: 192.168.6.63
  client:
    service-url:
      defaultZone: http://192.168.6.62:10086/eureka/,http://192.168.6.63:10086/eureka/
```

- 在启动微服务的时候，加入参数: spring.profiles.active（激活某个配置） 来读取对应的配置

### 2）其他微服务配置

- 除了Eureka注册中心以外，其他微服务配置都需要加入所有Eureka服务

  > 将之前的单个eureka微服务地址，改为多个eureka微服务的地址

```
#Eureka服务的注册信息
eureka:
  client:
    service-url:
      defaultZone: http://192.168.6.63:10086/eureka,http://192.168.6.62:10086/eureka
      # defaultZone: http://127.0.0.1:10086/eureka
  instance:
    lease-renewal-interval-in-seconds: 5 # 每隔5秒发送一次心跳
    lease-expiration-duration-in-seconds: 10 # 10秒不发送就过期
    prefer-ip-address: true
```

把代码提交到Gitlab中



## 4. 设计Jenkins集群项目的构建参数

### 1）安装Extended Choice Parameter插件

> 支持多选框

### 2）创建流水线项目



### 3）添加参数

> - 字符串参数：分支名称
>
> branch---->master

![image-20220316123954834](/images/posts/2022-3-9-Jenkins和Docker/image-20220316123954834.png)

> - 多选框：项目名称
>
> project_name----->check Boxes
>
> Choose Source for Value ----> value（tensquare_eureka_server@10086,tensquare_zuul@10020,tensquare_admin_server@9001,tensquare_gathering@9002）
>
> Choose Source for Default Value ----> Default value（tensquare_eureka_server@10086）

![image-20220316131054161](/images/posts/2022-3-9-Jenkins和Docker/image-20220316131054161.png)

![image-20220316131125423](/images/posts/2022-3-9-Jenkins和Docker/image-20220316131125423.png)

效果如下：

![image-20220316131202846](/images/posts/2022-3-9-Jenkins和Docker/image-20220316131202846.png)



> - 定义流水线

![image-20220316125831885](/images/posts/2022-3-9-Jenkins和Docker/image-20220316125831885.png)

## 5. 完成微服务构建镜像，上传私服（Jenkinsfile）

### 1）修改Jenkinsfile

```
//git凭证ID
def git_auth = "b632ed00-fc81-43c8-a746-5aa0673b2658"
//git的url地址
def git_url = "git@192.168.66.100:itheima_group/tensquare_back.git"
//镜像的版本号
def tag = "latest"
//Harbor的url地址
def harbor_url = "192.168.66.102:85"
//镜像库项目名称
def harbor_project = "tensquare"
//Harbor的登录凭证ID
def harbor_auth = "833d1a75-f3db-4aec-9cc4-75a77e423163"

node {
   //获取当前选择的项目名称
   def selectedProjectNames = "${project_name}".split(",")
   //获取当前选择的服务器名称
   def selectedServers = "${publish_server}".split(",")

   stage('拉取代码') {
      checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
   }
   stage('代码审查') {
        for(int i=0;i<selectedProjectNames.length;i++){
            //tensquare_eureka_server@10086
            def projectInfo = selectedProjectNames[i];
            //当前遍历的项目名称
            def currentProjectName = "${projectInfo}".split("@")[0]
            //当前遍历的项目端口
            def currentProjectPort = "${projectInfo}".split("@")[1]

            //定义当前Jenkins的SonarQubeScanner工具
            def scannerHome = tool 'sonar-scanner'
            //引用当前JenkinsSonarQube环境
            withSonarQubeEnv('sonarqube') {
                 sh """
                         cd ${currentProjectName}
                         ${scannerHome}/bin/sonar-scanner
                 """
            }
        }


   }
   stage('编译，安装公共子工程') {
      sh "mvn -f tensquare_common clean install"
   }
   stage('编译，打包微服务工程，上传镜像') {
       for(int i=0;i<selectedProjectNames.length;i++){
                 //tensquare_eureka_server@10086
                 def projectInfo = selectedProjectNames[i];
                 //当前遍历的项目名称
                 def currentProjectName = "${projectInfo}".split("@")[0]
                 //当前遍历的项目端口
                 def currentProjectPort = "${projectInfo}".split("@")[1]

                 sh "mvn -f ${currentProjectName} clean package dockerfile:build"

                 //定义镜像名称
                 def imageName = "${currentProjectName}:${tag}"

                 //对镜像打上标签
                 sh "docker tag ${imageName} ${harbor_url}/${harbor_project}/${imageName}"

                //把镜像推送到Harbor
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {

                    //登录到Harbor
                    sh "docker login -u ${username} -p ${password} ${harbor_url}"

                    //镜像上传
                    sh "docker push ${harbor_url}/${harbor_project}/${imageName}"

                    sh "echo 镜像上传成功"
                }

                //遍历所有服务器，分别部署
                for(int j=0;j<selectedServers.length;j++){
                       //获取当前遍历的服务器名称
                       def currentServerName = selectedServers[j]

                       //加上的参数格式：--spring.profiles.active=eureka-server1/eureka-server2
                       def activeProfile = "--spring.profiles.active="

                       //根据不同的服务名称来读取不同的Eureka配置信息
                       if(currentServerName=="master_server"){
                          activeProfile = activeProfile+"eureka-server1"
                       }else if(currentServerName=="slave_server"){
                          activeProfile = activeProfile+"eureka-server2"
                       }

                       //部署应用
                       sshPublisher(publishers: [sshPublisherDesc(configName: "${currentServerName}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/opt/jenkins_shell/deployCluster.sh $harbor_url $harbor_project $currentProjectName $tag $currentProjectPort $activeProfile", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])


                }

        }
   }
}
```

## 6. 完成微服务多服务器远程发布

### 1）配置远程部署服务器

- 拷贝公钥到远程服务器

```
ssh-copy-id 192.168.6.62
```

- 系统配置->添加远程服务器

### 2）修改Docker配置信任Harbor私服地址

```
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://3fc19s4g.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.6.62:85"]
}
```

重启Docker

### 3）添加参数

多选框：部署服务器

![image-20220316142749282](/images/posts/2022-3-9-Jenkins和Docker/image-20220316142749282.png)

![image-20220316142808854](/images/posts/2022-3-9-Jenkins和Docker/image-20220316142808854.png)

效果：

![image-20220316142838060](/images/posts/2022-3-9-Jenkins和Docker/image-20220316142838060.png)

### 4）修改Jenkinsfile构建脚本

```
//git凭证ID
def git_auth = "b632ed00-fc81-43c8-a746-5aa0673b2658"
//git的url地址
def git_url = "git@192.168.66.100:itheima_group/tensquare_back.git"
//镜像的版本号
def tag = "latest"
//Harbor的url地址
def harbor_url = "192.168.66.102:85"
//镜像库项目名称
def harbor_project = "tensquare"
//Harbor的登录凭证ID
def harbor_auth = "833d1a75-f3db-4aec-9cc4-75a77e423163"

node {
   //获取当前选择的项目名称
   def selectedProjectNames = "${project_name}".split(",")
   //获取当前选择的服务器名称
   def selectedServers = "${publish_server}".split(",")

   stage('拉取代码') {
      checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
   }
   stage('代码审查') {
        for(int i=0;i<selectedProjectNames.length;i++){
            //tensquare_eureka_server@10086
            def projectInfo = selectedProjectNames[i];
            //当前遍历的项目名称
            def currentProjectName = "${projectInfo}".split("@")[0]
            //当前遍历的项目端口
            def currentProjectPort = "${projectInfo}".split("@")[1]

            //定义当前Jenkins的SonarQubeScanner工具
            def scannerHome = tool 'sonar-scanner'
            //引用当前JenkinsSonarQube环境
            withSonarQubeEnv('sonarqube') {
                 sh """
                         cd ${currentProjectName}
                         ${scannerHome}/bin/sonar-scanner
                 """
            }
        }


   }
   stage('编译，安装公共子工程') {
      sh "mvn -f tensquare_common clean install"
   }
   stage('编译，打包微服务工程，上传镜像') {
       for(int i=0;i<selectedProjectNames.length;i++){
                 //tensquare_eureka_server@10086
                 def projectInfo = selectedProjectNames[i];
                 //当前遍历的项目名称
                 def currentProjectName = "${projectInfo}".split("@")[0]
                 //当前遍历的项目端口
                 def currentProjectPort = "${projectInfo}".split("@")[1]

                 sh "mvn -f ${currentProjectName} clean package dockerfile:build"

                 //定义镜像名称
                 def imageName = "${currentProjectName}:${tag}"

                 //对镜像打上标签
                 sh "docker tag ${imageName} ${harbor_url}/${harbor_project}/${imageName}"

                //把镜像推送到Harbor
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {

                    //登录到Harbor
                    sh "docker login -u ${username} -p ${password} ${harbor_url}"

                    //镜像上传
                    sh "docker push ${harbor_url}/${harbor_project}/${imageName}"

                    sh "echo 镜像上传成功"
                }

                //遍历所有服务器，分别部署
                for(int j=0;j<selectedServers.length;j++){
                       //获取当前遍历的服务器名称
                       def currentServerName = selectedServers[j]

                       //加上的参数格式：--spring.profiles.active=eureka-server1/eureka-server2
                       def activeProfile = "--spring.profiles.active="

                       //根据不同的服务名称来读取不同的Eureka配置信息
                       if(currentServerName=="master_server"){
                          activeProfile = activeProfile+"eureka-server1"
                       }else if(currentServerName=="slave_server"){
                          activeProfile = activeProfile+"eureka-server2"
                       }

                       //部署应用
                       sshPublisher(publishers: [sshPublisherDesc(configName: "${currentServerName}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/opt/jenkins_shell/deployCluster.sh $harbor_url $harbor_project $currentProjectName $tag $currentProjectPort $activeProfile", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])


                }

        }
   }
}
```



### 5）编写deployCluster.sh部署脚本

```
#! /bin/sh
#接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4
port=$5
profile=$6

imageName=$harbor_url/$harbor_project_name/$project_name:$tag

echo "$imageName"

#查询容器是否存在，存在则删除
containerId=`docker ps -a | grep -w ${project_name}:${tag} | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
	#停掉容器
	docker stop $containerId
	
	#删除容器
	docker rm $containerId
	echo "成功删除容器"
fi

#查询镜像是否存在，存在则删除
imageId=`docker images | grep -w $project_name | awk '{print $3}'`
if [ "$imageId" != "" ] ; then
	#删除镜像
	docker rmi -f $imageId
	echo "成功删除镜像"
fi

# 登录Harbor私服
docker login -u itcast -p Itcast123 $harbor_url
# 下载镜像
docker pull $imageName
# 启动容器
docker run -di -p $port:$port $imageName $profile
echo "容器启动成功"

```



### 6）集群效果



## 7. Nginx+Zuul集群实现高可用网关

![image-20220316143229277](/images/posts/2022-3-9-Jenkins和Docker/image-20220316143229277.png)

### 1）修改nginx配置

```
vi /etc/nginx/nginx.conf

upstream zuulServer{
	server 192.168.6.63:10020 weight=1;
	server 192.168.6.162:10020 weight=1;
}
server {
	listen 85 default_server;
	listen [::]:85 default_server;
	server_name _;
	root /usr/share/nginx/html;
	
	# Load configuration files for the default server block.
	include /etc/nginx/default.d/*.conf;
	
	location / {
		### 指定服务器负载均衡服务器
		proxy_pass http://zuulServer/;
}

```

> 重启Nginx：
>
> systemctl restart nginx

- 修改前端Nginx的访问地址

```
vi prod.env.js
'use strict'
module.exports = {
  NODE_ENV: '"production"',
  // BASE_API: '"http://localhost:10020"' // 管理员网关
  BASE_API: '"http://192.168.6.63:85"' // 管理员网关
  
}
```





































