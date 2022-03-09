---
layout: post
title: "Jenkins"
description: "分享"
tag: Jenkins
---

# 1、Jenkins介绍

![img](https://www.jenkins.io/images/logos/jenkins/jenkins.svg)

**Jenkins是一款流行的开源持续集成(Continuous Integration)工具，广泛用于项目开发，具有自动化构建、测试和部署等功能。**

官网: [http://jenkins-ci.org](http://jenkins-ci.org)

Jenkins的特征:

- 开源的Java语言开发持续集成工具，支持持续集成，持续部署。
- 易于安装部署配置:可通过yum安装,或下载war包以及通过docker容器等快速实现安装部署，可方便web界面配置管理。
- 消息通知及测试报告:集成RSS/E-mail通过RSs发布构建结果或当构建完成时通过e-mail通知，生成JUnit/TestNG测试报告。

- 分布式构建:支持Jenkins能够让多台计算机一起构建/测试。
- 文件识别: Jenkins能够跟踪哪次构建生成哪些jar，哪次构建使用哪个版本的jar等。
- 丰富的插件支持:支持扩展插件，你可以开发适合自己团队使用的工具，如git，svn，maven，docker等。

# 2、Jenkins安装和持续集成部署

## 1. 持续集成流程

1. 首先，开发人员每天进行代码提交，提交到Git仓库
2. 然后，Jenkins作为持续集成工具，使用Git工具到Git仓库拉取代码到集成服务器再配合JDK,Maven等软件完成代码编译，代码测试与审查，测试，打包等工作，在这个过程中每一步出错都重新再执行一次整个流程。
3. 最后，Jenkins把生成的jar或war包分发到测试服务器或者生产服务器，测试人员或用户就可以访问应用。



**环境列表**

| 作用域         | 系统/主机名         | IP           | 软件                                 |
| -------------- | ------------------- | ------------ | ------------------------------------ |
| 持续集成服务器 | CentOS7/k8s-master1 | 192.168.6.20 | Jenkins、JDK8、Maven、Git、SonarQube |
| 代码托管服务器 | CentOS7/k8s-node3   | 192.168.6.61 | Gitlab                               |
| 应用测试服务器 | CentOS7/k8s-node4   | 192.168.6.62 | JDK8、Tomcat                         |

环境准备：

```
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-node3
hostnamectl set-hostname k8s-node4

cat >> /etc/hosts << EOF
192.168.6.20 k8s-master1
192.168.6.61 k8s-node3
192.168.6.62 k8s-node4
EOF

ping k8s-master1
ping k8s-node3
ping k8s-node4
```

更换YUM源： 进入网站 [阿里巴巴开源镜像站](https://developer.aliyun.com/mirror/) 点击`centos`，根据步骤操作即可（建议提前安装好wget）

## 2. Gitlab介绍和安装

### 1、Gitlab介绍

![img](https://about.gitlab.com/images/icons/logos/slp-logo.svg)

官网：[https://about.gitlab.com](https://about.gitlab.com/)

GitLab是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的web服务。

GtLab和GitHub一样属于第三万基于GIl开风PDTFB的巨citLab是可以部署到自己的服务器上，数据库等一用户，任意提交你的代码，添加SSHKey等等。不同的是，**GitLab是可以部署到自己的服务器上，数据库等一切信息都掌握在自己手上，适合团队内部协作开发**，你总不可能把团队内部的智慧总放在别人的服务器上吧?简单来说可把GitLab看作个人版的GitHub。

### 2、Gitlab安装（代码托管服务器）



```
# 1. 安装相关依赖
yum -y install policycoreutils openssh-server openssh-clients postfix policycoreutils-python

# 2. 启动ssh服务
systemctl enable sshd && sudo systemctl start sshd

# 3. 设置postfix开机自启，并支持gitlab发信功能
systemctl enable postfix && systemctl start postfix

# 4. 开放ssh以及http服务，重新加载防火墙（如果防火墙关闭，则不用）
# 查看防火墙
systemctl status firewalld
# 防火墙开放服务（如果防火墙关闭，则不用）
firewall-cmd --add-service=ssh --permanent
firewall-cmd --add-service=http --permanent
firewall-cmd --reload	# 重新加载配置生效

# 5. 下载Gitlab存储库
# 清华大学开源网站（https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/）
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-14.8.0-ce.0.el7.x86_64.rpm
# 安装
rpm -i gitlab-ce-14.8.0-ce.0.el7.x86_64.rpm

# 6. 修改Gitlab配置
vi /etc/gitlab/gitlab.rb
# 修改访问地址和端口
external_url 'http://192.168.6.61'
nginx['listen_port'] = 82

# 7. 重载配置以及启动Gitlab
gitlab-ctl reconfigure
gitlab-ctl restart

# 8. 把端口添加到防火墙
firewalld-cmd --zone=public --add-port=82/tcp --permanent
firewalld-cmd --reload

```



根据提示获取密码信息：

```
cat /etc/gitlab/initial_root_password 
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: GIprGS+8LS4CsPePzCusdeo05ahCPh69wwkzjuSs74E=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```



启动后访问：

![image-20220228153449616](/images/posts/2022-3-3-Jenkins/image-20220228153449616.png)



**重置密码：**

可以使用 Rake 任务、Rails 控制台或 [用户 API](https://docs.gitlab.com/ee/api/users.html#user-modification)重置用户密码。

```
# 使用Rake 为root用户 重置密码（密码：root1234）
sudo gitlab-rake "gitlab:password:reset[root]"

```

![image-20220228160823394](/images/posts/2022-3-3-Jenkins/image-20220228160823394.png)



刷新页面重新登陆

### 3、Gitlab添加组、创建用户、创建项目

#### 1）创建组

使用管理员root创建组，一个组里面可以有多个项目分支，可以将开发添加到组里面进行设置权限，不同的组就是公司不同的开发项目或者服务模块，不同的组添加不同的开发即可实现对开发设置权限的管理

![image-20220228161548522](/images/posts/2022-3-3-Jenkins/image-20220228161548522.png)

![image-20220228161904859](/images/posts/2022-3-3-Jenkins/image-20220228161904859.png)

![image-20220228162125247](/images/posts/2022-3-3-Jenkins/image-20220228162125247.png)

可以根据`New project` 创建项目

#### 2）创建项目

![image-20220228162404868](/images/posts/2022-3-3-Jenkins/image-20220228162404868.png)

创建成功

![image-20220228162546573](/images/posts/2022-3-3-Jenkins/image-20220228162546573.png)

#### 3）创建用户

创建用户并将用户分配到这个组里

![image-20220228162753613](/images/posts/2022-3-3-Jenkins/image-20220228162753613.png)

创建用户

> 密码后期配置
>
> Projects limit：管理的项目上限数
>
> 权限：Regular（普通用户），访问自己的项目和组
>
> ​			Administrator（管理员），可以无限访问所有组、项目、用户和功能。



![image-20220228163356691](/images/posts/2022-3-3-Jenkins/image-20220228163356691.png)

设置密码：zhangsan123

![image-20220228163601248](/images/posts/2022-3-3-Jenkins/image-20220228163601248.png)

#### 4）给组添加成员（添加用户），并分配权限

![image-20220228164018329](/images/posts/2022-3-3-Jenkins/image-20220228164018329.png)

添加zhangsan用户并分配相应权限

> 5中权限说明：
>
> Gust（访客）：可以创建issue、发表评论、不能读写版本库
>
> Reporter：可以克隆代码，不能提交，QA、PM可以赋予这个权限（一般项目经理需要）
>
> Developer：可以克隆代码、开发、提交、 push，（普通开发可以赋予这个权限）
>
> Maintainer：可以创建项目、添加tag、保护分支、添加项目成员、编辑项目，（核心开发可以赋予这个权限）
>
> Owner：可以设置项目访问权限-Visibility Level、删除项目、迁移项目、管理组成员，（开发组组长可以赋予这个权限）

![image-20220228164613398](/images/posts/2022-3-3-Jenkins/image-20220228164613398.png)

退出使用zhangsan用户登陆，第一次登陆需要修改密码（这里保持密码不变：zhangsan/zhangsan123）

![image-20220228165009780](/images/posts/2022-3-3-Jenkins/image-20220228165009780.png)

登陆

![image-20220228165142273](/images/posts/2022-3-3-Jenkins/image-20220228165142273.png)

## 3. 源码上传到Gitlab

项目构建请移步我的博客：[geray-zsg.github.io](geray-zsg.github.io)

### 1）IDEA工具准备好一个简单的web项目

> 项目中的web.xml中一定要清理错误信息，否则后续上传远程tomcat服务会失败
>
> ```
> 03-Mar-2022 19:11:28.773 严重 [http-nio-8081-exec-5] org.apache.catalina.core.StandardContext.startInternal 一个或多个listeners启动失败，更多详细信息查看对应的容器日志文件
> 03-Mar-2022 19:11:28.773 严重 [http-nio-8081-exec-5] org.apache.catalina.core.StandardContext.startInternal 由于之前的错误，Context[/web-demo-1.0-SNAPSHOT]启动失败
> 
> ```

项目结构如下：

![image-20220303153006113](/images/posts/2022-3-3-Jenkins/image-20220303153006113.png)

### 2）推送项目到Gitlab

> 点击项目 ---> 点击VCS（版本控制服务） ---> Enable Version Control Integration ---> 选择Git（最终在项目的根目录下创建了一个本地仓库）

![image-20220303153142756](/images/posts/2022-3-3-Jenkins/image-20220303153142756.png)



> 上一步选择Git，并下载完成之后继续
>
> 1、将本地代码添加代本地仓库暂存区：右击项目 ---> 选择Git ---> Add
>
> 2、将本地代码提交到本地仓库：右击项目 ---> 选择Git ---> Commit Directory  （Commit Message，提交的描述信息）
>
> 3、添加远程仓库：右击项目 ---> 选择Git ---> Repository ---> Remots
>
> 4、推送本地仓库代码到Gitlab：右击项目 ---> 选择Git ---> Repository ---> Push ---> 选择刚才添加的远程仓库地址即可

**提交到暂本地仓库：**

![image-20220303155632107](/images/posts/2022-3-3-Jenkins/image-20220303155632107.png)

提交

![image-20220303155456099](/images/posts/2022-3-3-Jenkins/image-20220303155456099.png)

点击`9:Git`可以查看git的日志信息：

![image-20220303155600443](/images/posts/2022-3-3-Jenkins/image-20220303155600443.png)

**添加远程仓库：**

> 3、添加远程仓库：右击项目 ---> 选择Git ---> Repository ---> Remots
>
> 由于我上面部署Gitlab时修改了端口号为82
>
> 所以这里的地址应该是：http://192.168.6.61:82/group1/web_demo.git
>
> 账号密码，暂时使用：zhangsan/zhangsan123

![image-20220303162651350](/images/posts/2022-3-3-Jenkins/image-20220303162651350.png)

**推送到Gitlab：**

> 4、推送本地仓库代码到Gitlab：右击项目 ---> 选择Git ---> Repository ---> Push ---> 选择刚才添加的远程仓库地址即可

![image-20220303162958953](/images/posts/2022-3-3-Jenkins/image-20220303162958953.png)

刷新Gitlab：查看已经提交成功

![image-20220303163047531](/images/posts/2022-3-3-Jenkins/image-20220303163047531.png)

## 4. 持续集成环境-Jenkins安装

```
# 1、添加存储库
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# 2、安装（下载路径：https://www.jenkins.io/zh/download）（war包下载位置：http://mirrors.jenkins.io/war/）
yum install epel-release -y # 安装依赖（提供“daemonize”的存储库）

# yum install java-11-openjdk-devel -y # 安装jdk
# yum remove java-11-openjdk-devel -y

yum install java-1.8.*-openjdk-devel -y

yum install jenkins -y # 安装Jenkins

# 3、修改Jenkins配置
vi /etc/sysconfig/jenkins
# 修改内容如下
JENKINS_USER="root"
JENKINS_PORT="8888"

# 4、启动Jenkins
systemctl start jenkins
systemctl status jenkins

# 5、添加防火墙策略或者关闭防火墙
systemctl stop firewalld 
systemctl disabled firewalld 

# 6、浏览器访问
http://192.168.6.20:8888
```

根据提示信息获取密码：

![image-20220301140231655](/images/posts/2022-3-3-Jenkins/image-20220301140231655.png)

**跳过插件安装：**

> Jenkins插件需要连接默认官方下载，速度非常慢，而且经常失败，所以暂时跳过插件安装

![image-20220301140712556](/images/posts/2022-3-3-Jenkins/image-20220301140712556.png)

创建新的管理员账号：geray/123456（默认有一个Admin账号密码就是上面解锁时用到的）

![image-20220301140929217](/images/posts/2022-3-3-Jenkins/image-20220301140929217.png)

一路辖下去默认基本就搞定了

## 5. 持续集成环境-Jenkins插件管理

Jenkins本身不提供很多功能，我们可以通过使用插件来满足我们的使用。例如从Gitlab拉取代码，使用Maven构建项目等功能需要依靠插件完成。接下来演示如何下载插件。

### 1、修改插件下载地址（清华大学为例）

Jenkins国外官方插件地址下载速度非常慢，所以可以修改为国内插件地址：

> 清华大学：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/
>
> 阿里云：https://mirrors.aliyun.com/jenkins/updates/
>
> 华为云：https://mirrors.huaweicloud.com/jenkins/updates/





**1、复制下载地址**：

![image-20220301145947474](/images/posts/2022-3-3-Jenkins/image-20220301145947474.png)

**2、修改地址：**

> Jenkins ---> Manage Jenkins --->  Manage Plugins，点击Advanced ，修改`Update Site`的URL，提交保存

![image-20220301150355940](/images/posts/2022-3-3-Jenkins/image-20220301150355940.png)



> Jenkins默认的开发目录(/var/lib/jenkins/)，的updates目录下有个default.json文件，记录了默认官方下载地址（国外）

**3、浏览器输入重启指令：**

```
http://192.168.6.20:8888/restart
```

并使用`geray/123456`账号登陆

### 2、下载中文汉化插件

> Jenkins ---> Manage Jenkins --->  Manage Plugins，点击Available，搜索“Chinese”
>
> 也可以直接通过[https://updates.jenkins-ci.org/download/plugins/](https://links.jianshu.com/go?to=https%3A%2F%2Fupdates.jenkins-ci.org%2Fdownload%2Fplugins%2F) 直接下载需要的插件版本

![image-20220301151854523](/images/posts/2022-3-3-Jenkins/image-20220301151854523.png)

安装并重启，重置之后即可看到已被汉化的效果

## 6. 持续集成环境-Jenkins用户权限管理

我们可以利用Role-based Authorization Strategy插件来管理enkins用户权限

### 1、安装Role-based Authorization Strategy插件

> Jenkins ---> Manage Jenkins --->  Manage Plugins，点击Available，搜索“Role-based Authorization Strategy”

![image-20220301152321825](/images/posts/2022-3-3-Jenkins/image-20220301152321825.png)

### 2、开启权限全局安全配置

> Manage Jenkins --->  Configure Global Security  ---> 授权策略（默认`Logged-in users can do anything`：任何人可以做任何事情） ---> 选择`Role-Based Strategy`  ---> 保存

![image-20220301155727575](/images/posts/2022-3-3-Jenkins/image-20220301155727575.png)

### 3、创建角色

> Manage Jenkins --->  Manage and Assign Roles --->  Manage Roles （管理角色）
>
> Global roles（全局角色）：管理Jenkins的管理员
>
> Item roles（项目角色）：针对项目分配的角色
>
> Node roles（节点角色）：主从情况下

创建全局角色（baseRole）并分配权限（Overall：Read）

![image-20220301161141784](/images/posts/2022-3-3-Jenkins/image-20220301161141784.png)

创建项目角色（role1），确保该角色能够访问以`geray`开头的项目（geray.*），并分配权限（所有权限）

![image-20220301161700174](/images/posts/2022-3-3-Jenkins/image-20220301161700174.png)

创建第二个项目角色（role2），确保该角色能够访问以`demo`开头的项目（demo.*），并分配权限（所有权限）

![image-20220301161857633](/images/posts/2022-3-3-Jenkins/image-20220301161857633.png)

### 4、创建用户

> Manage Jenkins --->  Manage Users（管理用户） --->  新建用户
>
> 创建两个用户：tom/123456  和 jack/123456

![image-20220301165317129](/images/posts/2022-3-3-Jenkins/image-20220301165317129.png)

### 5、给用户分配角色

> Manage Jenkins --->  Manage and Assign Roles  ---> （分配角色）
>
> Global roles（全局角色）：管理Jenkins的管理员
>
> Item roles（项目角色）：针对项目分配的角色
>
> Node roles（节点角色）：主从情况下

1. 给tom用户分配全局角色（Global roles）的基础角色（baseRole，之前创建的角色，具有登陆权限）

![image-20220301170018891](/images/posts/2022-3-3-Jenkins/image-20220301170018891.png)

2. 给tom用户分配项目角色（Item roles）的role1角色（可以访问geray开头的项目）



> 同理给jack用户分配Global roles/baseRole 和 Item roles/role2

### 6、创建项目

> 新建Item ----> 分别创建一个geray和demo开头的项目并使用自由风格（其他保持默认）

![image-20220301170816343](/images/posts/2022-3-3-Jenkins/image-20220301170816343.png)

创建成功后：

![image-20220301171155213](/images/posts/2022-3-3-Jenkins/image-20220301171155213.png)

测试：

> 分别使用tom和jack登陆，只能看到相关的项目即ok了（tom ---> geray.*   |    jack ---->  demo.*）

## 7. 持续集成环境-Jenkins凭证管理

凭据可以用来存储需要密文保护的数据库密码、Gitlab密码信息、Docker私有仓库密码等，以便Jenkins可以和这些第三方的应用进行交互。

### 1、安装Credentials Binding插件

要在Jenkins使用凭证管理功能，需要安装`Credentials Binding`插件

> 通过[https://updates.jenkins-ci.org/download/plugins/](https://links.jianshu.com/go?to=https%3A%2F%2Fupdates.jenkins-ci.org%2Fdownload%2Fplugins%2F) 直接下载需要的插件版本

![image-20220301182923969](/images/posts/2022-3-3-Jenkins/image-20220301182923969.png)



通过web导入：

![image-20220301183422971](/images/posts/2022-3-3-Jenkins/image-20220301183422971.png)

### 2、添加凭据

> Manage Jenkins --->  Manage Credentials 

![image-20220301185321092](/images/posts/2022-3-3-Jenkins/image-20220301185321092.png)

点击全局

![image-20220301185509531](/images/posts/2022-3-3-Jenkins/image-20220301185509531.png)

> 支持5中类型：
>
> Username with password：用户名密码（常见）  ----->   对应`Clone with SSH`
>
> SSH  Username with private key：常用于SSH免密登陆的情况下（常见）  ----->   对应`Clone with HTTP`
>
> Secret file：秘钥文件
>
> Secret text：秘钥文本
>
> Certificate：证书类型（少见）
>
> 常用的凭证类型有: Username withjpassword(用户密码）和SSH Username writh private key (SSH密钥)

#### 1）安装Git插件和Git工具

为了让Jenkins支持从Gitlab拉取源码，需要安装Git插件以及在CentOS7上安装GitGit插件安装:

![image-20220301193222923](/images/posts/2022-3-3-Jenkins/image-20220301193222923.png)

**Jenkins环境服务器**上安装Git工具

```
yum -y install git
# 查看git版本
git --version
```

#### 2）创建用户名密码类型凭据

**1）创建凭据**

> Jenkins->Manage Jenkins->Manage Credentials  ->全局凭证->添加凭据

![image-20220301204300728](/images/posts/2022-3-3-Jenkins/image-20220301204300728.png)

选择"Username with password"，输入Gitlab的用户名和密码，点击"确定"。创建完成之后：

![image-20220301204335464](/images/posts/2022-3-3-Jenkins/image-20220301204335464.png)

**2）测试凭据是否可用**

创建一个FreeStyle项目：新建Item->FreeStyle Project->确定

![image-20220303163834491](/images/posts/2022-3-3-Jenkins/image-20220303163834491.png)

找到"源码管理"->"Git"，在Repository URL复制Gitlab中的项目URL

![image-20220301204807651](/images/posts/2022-3-3-Jenkins/image-20220301204807651.png)



配置新建的项目

![image-20220301204945543](/images/posts/2022-3-3-Jenkins/image-20220301204945543.png)

选择源码管理为Git，并粘贴复制的UTL（http://192.168.6.61:82/group1/web_demo.git）；注意端口号

![image-20220301205200408](/images/posts/2022-3-3-Jenkins/image-20220301205200408.png)

这时会报错说无法连接仓库！在Credentials选择刚刚添加的凭证就不报错啦

![image-20220301205254620](/images/posts/2022-3-3-Jenkins/image-20220301205254620.png)

保存配置后，点击构建”Build Now“ 开始构建项目

![image-20220301205402316](/images/posts/2022-3-3-Jenkins/image-20220301205402316.png)

点击test01项目，构建历史中点击最新的一次构建-----> 控制台输出

![image-20220303164122655](/images/posts/2022-3-3-Jenkins/image-20220303164122655.png)

查看/var/lib/jenkins/workspace/目录，发现已经从Gitlab成功拉取了代码到Jenkins中。

![image-20220303164242424](/images/posts/2022-3-3-Jenkins/image-20220303164242424.png)

#### 3）SSH秘钥类型

![image-20220301210105735](/images/posts/2022-3-3-Jenkins/image-20220301210105735.png)

1）使用root用户生成公钥和私钥

```
ssh-keygen -t rsa
```

> **在/root/.ssh/目录保存了公钥和使用**
>
> id_rsa：私钥文件 
>
> id_rsa.pub：公钥文件

2）把生成的公钥放在Gitlab中

> 以root账户登录->点击头像->Edit profile（或者Settings）->SSH Keys
>
> 复制刚才id_rsa.pub文件的内容到这里，点击"Add Key"

![image-20220301210746496](/images/posts/2022-3-3-Jenkins/image-20220301210746496.png)

3）在Jenkins中添加凭证，配置私钥 在Jenkins添加一个新的凭证，类型为"SSH Username with private key"，把刚才生成私有文件内容复 制过来

![image-20220301210934308](/images/posts/2022-3-3-Jenkins/image-20220301210934308.png)

Username填写root，让后复制私有内容

![image-20220301211037897](/images/posts/2022-3-3-Jenkins/image-20220301211037897.png)

4）测试凭证是否可用 

> 新建"test02"项目->源码管理->Git，
>
> 这次要使用Gitlab的SSH连接，并且选择SSH凭证

![image-20220301211157924](/images/posts/2022-3-3-Jenkins/image-20220301211157924.png)

复制Gitlab项目的SSH

![image-20220301211237867](/images/posts/2022-3-3-Jenkins/image-20220301211237867.png)



![image-20220301211348547](/images/posts/2022-3-3-Jenkins/image-20220301211348547.png)

同样尝试构建项目，如果代码可以正常拉取，代表凭证配置成功

![image-20220303164415204](/images/posts/2022-3-3-Jenkins/image-20220303164415204.png)

## 8. 持续集成环境-Maven安装和配置

在Jenkins集成服务器上，我们需要安装Maven来编译和打包项目。

官方下载：https://maven.apache.org/download.cgi



**1）安装Maven**

先上传Maven软件到`Jenkins服务器`

```
tar -xf apache-maven-3.6.1-bin.tar.gz 

mkdir -p /opt/maven 

mv apache-maven-3.6.1 /opt/apache-maven-3.6.1 
```

**2）配置环境变量**

```
vi /etc/profile
# 配置Maven如下
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export MAVEN_HOME=/opt/apache-maven-3.6.1
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin

# 配置生效
source /etc/profile 
# 查找Maven版本
mvn -v 
```

**3）全局工具配置关联JDK和Maven**

> Manage Jenkins ---> Global Tool Configuration ---->  JDK  ---->  新增JDK，配置如下（取消自动安装：Install automatically）：

![image-20220302093158072](/images/posts/2022-3-3-Jenkins/image-20220302093158072.png)

应用、保存



> Jenkins  --->  Global Tool Configuration   ---->   Maven  ----->   新增Maven，配置如下（取消自动安装：Install automatically）：

![image-20220303165003749](/images/posts/2022-3-3-Jenkins/image-20220303165003749.png)

应用、保存

**4）添加Jenkins全局变量**

原理和linux配置环境变量一样，让Jenkins感知jdk和maven的命令

> Manage Jenkins   ---->    Configure System   ---->   Global Properties（全局属性） --->  Environment variables  ----> 新增 
>
> 添加三个全局变量 JAVA_HOME、M2_HOME、PATH+EXTRA
>
> ```
>  JAVA_HOME=/usr/lib/jvm/java-11-openjdk
>  M2_HOME=/opt/maven
>  PATH+EXTRA=$M2_HOME/bin
> ```

![image-20220303165204067](/images/posts/2022-3-3-Jenkins/image-20220303165204067.png)

应用、保存



**5）修改Maven的settings.xml**

> 可以参考我的博客maven与IDEA构建：[geray-zsg.github.io](https://geray-zsg.github.io/)

```
# 创建本地仓库目录
mkdir /root/repo 

vi /opt/maven/conf/settings.xml
# 修改为本地仓库地址
<localRepository>/root/repo</localRepository>

# 修改为阿里云私服地址
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

```

修改本地仓库地址

![image-20220302095016998](/images/posts/2022-3-3-Jenkins/image-20220302095016998.png)

添加阿里云地址：

![image-20220303165612027](/images/posts/2022-3-3-Jenkins/image-20220303165612027.png)



**6）测试Maven是否配置成功** （构建：将源码编译打包）

配置项目测试（这里使用test01项目）

> Jenkins ---> test01 ---> 配置 ---> 构建 --->  增加构建步骤 --->   Execute shell
>
> 命令：`mvn clean package`

![image-20220303165701319](/images/posts/2022-3-3-Jenkins/image-20220303165701319.png)

应用、保存

> Build Now （构建） ----> 构建历史 ----> 控制台输出
>
> 或者查看本地仓库是否有文件生成

![image-20220303170108357](/images/posts/2022-3-3-Jenkins/image-20220303170108357.png)



> 同样的方式测试test02项目

## 9. 持续集成环境-Tomcat安装和配置

### 1、安装Tomcat

把Tomcat压缩包上传到192.168.6.62服务器（这里使用apache-tomcat-9.0.59版本）

```
# 安装jdk
yum install java-1.8.0-openjdk* -y


# 解压tomcat，并放到/opt下
```

关闭防火墙

```
systemctl stop firewalld
```

### 2、配置Tomcat用户角色权限

Jenkins部署项目到Tomcat服务器，需要用到Tomcat的用户，所以修改tomcat以下配置， 添加用户及权限

```xml
vi /opt/apache-tomcat-9.0.59/conf/tomcat-users.xml

<role rolename="tomcat"/>
<role rolename="role1"/>
<role rolename="manager-script"/>
<role rolename="manager-gui"/>
<role rolename="manager-status"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<user username="tomcat" password="tomcat" roles="manager-gui,manager-script,tomcat,admin-gui,admin-script"/>

```

> 用户和密码都是：tomcat 

注意：为了能够让刚才配置的用户登录到Tomcat，还需要修改以下配置（注释掉一下内容）

```
vi /opt/apache-tomcat-9.0.59/webapps/manager/META-INF/context.xml

<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

### 3、重启Tomcat，访问测试

访问并使用用户登陆测试，可以登陆就ook了

![image-20220303180132716](/images/posts/2022-3-3-Jenkins/image-20220303180132716.png)

# 3、Jenkins项目构建

Jenkins中自动构建项目的类型有很多，常用的有以下三种：

- 自由风格软件项目（FreeStyle Project） 
- Maven项目（Maven Project） 
- 流水线项目（Pipeline Project） 

每种类型的构建其实都可以完成一样的构建过程与结果，只是在操作方式、灵活度等方面有所区别，在 实际开发中可以根据自己的需求和习惯来选择。

> **个人推荐使用流水线类型，因为灵活度非常高**

## 1、自由风格项目构建

创建一个自由风格项目来完成项目的集成过程：

> 拉取代码->编译->打包->部署

### 1）拉取代码

创建自由风格项目，并使用SSH方式拉取

![image-20220303181047764](/images/posts/2022-3-3-Jenkins/image-20220303181047764.png)

拉取方式（http也可以，随便）

![image-20220303181225824](/images/posts/2022-3-3-Jenkins/image-20220303181225824.png)

### 2）编译打包

> 构建  --->  增加构建步骤  ----> Executor shell

```
echo "开始编译打包"
mvn clean package
echo "结束编译打包"
```

构建完成可以看到在工作目录中已经生成了相关war包等信息

![image-20220303181937182](/images/posts/2022-3-3-Jenkins/image-20220303181937182.png)

### 3）部署

把项目部署到远程的Tomcat里面

**1）安装 Deploy to container插件** 

Jenkins本身无法实现远程部署到Tomcat的功能，需要安装`Deploy to container`插件实现

![image-20220303182242731](/images/posts/2022-3-3-Jenkins/image-20220303182242731.png)

**2）添加Tomcat用户凭据**

![image-20220303183345626](/images/posts/2022-3-3-Jenkins/image-20220303183345626.png)

**3）添加构建后操作以及**

> 构建后操作  ---->   增加构建后操作步骤  --->  `Deploy war/ear to a container`

![image-20220303183544042](/images/posts/2022-3-3-Jenkins/image-20220303183544042.png)

最后点击构建

并在tomcat查看和访问，并且能够正常访问

![image-20220303192529572](/images/posts/2022-3-3-Jenkins/image-20220303192529572.png)

### 4）改动代码后的持续集成

1）IDEA中源码修改并提交到gitlab 

2）在Jenkins中项目重新构建 

3）访问Tomcat

> 修改代码 ---->  提交到仓库  ---->   Gitlab查看是否提交成功 ---->  jenkins上重新构建项目（ok）

## 2、Maven项目构建

### 1）安装Maven Integration插件

![image-20220304090552868](/images/posts/2022-3-3-Jenkins/image-20220304090552868.png)

### 2）创建Maven项目

![image-20220304090710357](/images/posts/2022-3-3-Jenkins/image-20220304090710357.png)

### 3）配置项目

> 源码管理 ---->  git  

![image-20220304091656103](/images/posts/2022-3-3-Jenkins/image-20220304091656103.png)



> build  --->  指定pom.xml文件和maven指令（只写参数，不用写mvn命令）

![image-20220304090933984](/images/posts/2022-3-3-Jenkins/image-20220304090933984.png)



> 构建后操作 --->   增加构建后操作步骤   ---->   Deploy war/ear to a container

![image-20220304092702272](/images/posts/2022-3-3-Jenkins/image-20220304092702272.png)

> buil now （构建并访问）

![image-20220304093300622](/images/posts/2022-3-3-Jenkins/image-20220304093300622.png)

## 3、Pipeline流水线项目构建

### 1）Pipeline简介

Pipeline，简单来说，就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点 的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。

**优点：**

- 代码：Pipeline以代码的形式实现，通常被检入源代码控制，使团队能够编辑，审查和迭代其传送流 程。
- 持久：无论是计划内的还是计划外的服务器重启，Pipeline都是可恢复的。 
- 可停止：Pipeline可接 收交互式输入，以确定是否继续执行Pipeline。 
- 多功能：Pipeline支持现实世界中复杂的持续交付要 求。它支持fork/join、循环执行，并行执行任务的功能。 
- 可扩展：Pipeline插件支持其DSL的自定义扩 展 ，以及与其他插件集成的多个选项。

**如何创建 Jenkins Pipeline呢？**

- Pipeline 脚本是由 Groovy 语言实现的，但是我们没必要单独去学习 Groovy 
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法 
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一 个 Jenkinsfile 脚本文件放入项目源码库中（一般我们都推荐在 Jenkins 中直接从源代码控制(SCM) 中直接载入 Jenkinsfile Pipeline 这种方法）。

### 2）安装Pipeline插件

> Manage Jenkins -----> Manage Plugins  ----> 可选插件  ---> 搜索Pipeline

![image-20220304094318193](/images/posts/2022-3-3-Jenkins/image-20220304094318193.png)

- 安装插件后，创建项目的时候多了“流水线”类型

### 3）Pipeline语法快速入门 - Declarative声明式

1. 构建流水线项目

![image-20220304094656602](/images/posts/2022-3-3-Jenkins/image-20220304094656602.png)

2. 流水线 --->  脚本  --->  Hello World

![image-20220304094951332](/images/posts/2022-3-3-Jenkins/image-20220304094951332.png)

```
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

> stages：代表整个流水线的所有执行阶段。通常stages只有1个，里面包含多个stage
>
> stage：代表流水线中的某个阶段，可能出现n个。一般分为拉取代码，编译构建，部署等阶段。 
>
> steps：代表一个阶段内需要执行的逻辑。steps里面是shell脚本，git拉取代码，ssh远程发布等任意内容。

3. 编写一个简单声明式Pipeline，如下：

```
pipeline {
    agent any

    stages {
        stage('拉取代码') {
            steps {
                echo '拉取代码'
            }
        }
        stage('编译构建') {
            steps {
                echo '编译构建'
            }
        }
        stage('项目部署') {
            steps {
                echo '项目部署'
            }
        }    
    }
}
```

4. 应用 --->  保存 ---->  build now ；结果如下：

![image-20220304101746990](/images/posts/2022-3-3-Jenkins/image-20220304101746990.png)

### 4）Pipeline语法快速入门 - Scripted Pipeline脚本式

上面第二步时选择 “Scripted Pipeline”，其他步骤一样

![image-20220304102340090](/images/posts/2022-3-3-Jenkins/image-20220304102340090.png)

根据生成的模板编写一个简单的脚本式pipeline

```
node {
    def mvnHome
    stage('拉取代码') {
       echo '拉取代码'
    }  
    stage('编译构建') {
       echo '编译构建'
    }  
    stage('项目部署') {
       echo '项目部署'
    }  
}
```

> Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行 环境，后续讲到Jenkins的Master-Slave架构的时候用到。 
>
> Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如： Build、Test、Deploy，Stage 是一个逻辑分组的概念。 
>
> Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像， 由各类 Jenkins 插件提供，比如命令：sh ‘make’，就相当于我们平时 shell 终端中执行 make 命令 一样。

构建结果和声明式一样！

![image-20220304102932109](/images/posts/2022-3-3-Jenkins/image-20220304102932109.png)

### 5）Pipeline - Declarative声明式构建pipeline01项目

每个片段都可以单独进行构建测试，确保都是正确的！

**1. 生成代码片段 - 拉取代码部分**

> pipeline01  --->  配置 ---->  流水线  ---->  流水线语法  ---> 片段生成器  ---> 示例步骤选择（checkout: Check out from version control）

![image-20220304104107257](/images/posts/2022-3-3-Jenkins/image-20220304104107257.png)

添加相关信息并生成代码：

![image-20220304104813734](/images/posts/2022-3-3-Jenkins/image-20220304104813734.png)



将结果代码嵌套到“代码拉取”中

**2. 生成代码片段 - 编译构建部分**

> 这个可以手写，也可以生成
>
> 这里使用代码生成
>
> 片段生成器  ---> 示例步骤选择（sh: Shell Script）
> Shell Script中写入指令：mvn clean package

![image-20220304105245702](/images/posts/2022-3-3-Jenkins/image-20220304105245702.png)



**3. 生成代码片段 - 项目部署部分**

> 片段生成器  ---> 示例步骤选择（deploy: Deploy war/ear to a container） -->  没有选项说明没有安装插件
> 参数和maven构建的“构建后操作”一样
>
> - 容器可以增加多台；同时部署

![image-20220304110016234](/images/posts/2022-3-3-Jenkins/image-20220304110016234.png)

**4. 最终组合在一起**

```
pipeline {
    agent any

    stages {
        stage('拉取代码') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'c704a539-2788-4929-bb75-dfcb4dc6054b', url: 'git@192.168.6.61:group1/web_demo.git']]])
            }
        }
        stage('编译构建') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('项目部署') {
            steps {
                deploy adapters: [tomcat9(credentialsId: '6ab790c8-c190-4fd2-a16f-62d48f84b03e', path: '', url: 'http://192.168.6.62:8080/')], contextPath: null, war: 'target/*.war'
            }
        }    
    }
}
```

![image-20220304110118653](/images/posts/2022-3-3-Jenkins/image-20220304110118653.png)

应用、保存，构建测试

![image-20220304110419232](/images/posts/2022-3-3-Jenkins/image-20220304110419232.png)

# 4、管理Jenkinsfile脚本文件

Jenkins的UI界面编写Pipeline代码，这样不方便脚本维护，建议把Pipeline脚本放 在项目中（一起进行版本控制）

## 1. 在项目根目录建立Jenkinsfile文件，把内容复制到该文件中

![image-20220304111110455](/images/posts/2022-3-3-Jenkins/image-20220304111110455.png)

## 2. 将Jenkinsfile文件提交到Gitlab仓库

> 右键Jenkinsfile文件 ---> git --->  commit file

![image-20220304111341563](/images/posts/2022-3-3-Jenkins/image-20220304111341563.png)

> 右键Jenkinsfile文件 ---> git --->  repository ---> push

![image-20220304111552747](/images/posts/2022-3-3-Jenkins/image-20220304111552747.png)

## 3. 配置Jenkins项目的流水线  -- Pipeline script from SCM

> 定义选择“Pipeline script from SCM”

![image-20220304112045769](/images/posts/2022-3-3-Jenkins/image-20220304112045769.png)

构建测试

![image-20220304112141646](/images/posts/2022-3-3-Jenkins/image-20220304112141646.png)

# 5、Jenkins项目构建细节

## 1. 常见的构建触发器

Jenkins内置4种构建触发器： 

- 触发远程构建 
- 其他工程构建后触发（Build after other projects are build）
- 定时构建（Build periodically） 
- 轮询SCM（Poll SCM）

> Jenkins --->  <项目>   ---->  配置 ---->   构建触发器

![image-20220304141647181](/images/posts/2022-3-3-Jenkins/image-20220304141647181.png)

### 1、触发远程构建

使用pipeline02项目演示触发远程构建

> 配置触发器

![image-20220304142449878](/images/posts/2022-3-3-Jenkins/image-20220304142449878.png)

应用、保存；并触发

![image-20220304142752520](/images/posts/2022-3-3-Jenkins/image-20220304142752520.png)

触发构建的URL：<JENKINS_URL>/job/项目名/build?token=token

​				例如：http://192.168.6.20:8888//job/pipeline02/build?token=token=11111111

### 2、其他工程构建后触发（Build after other projects are build）

#### 1）创建一个工程用于触发构建（前置工程：pre_job）

> 自由风格即可

![image-20220304144119587](/images/posts/2022-3-3-Jenkins/image-20220304144119587.png)



#### 2）配置pipeline01为需要触发的工程

> 构建触发器  ---->  Build after other projects are built
>
> 选择关注的前置工程

![image-20220304144322126](/images/posts/2022-3-3-Jenkins/image-20220304144322126.png)

应用保存，并观察触发行为（没有，等构建pre_job工程时，将会触发pipeline01的构建）

#### 3）构建pre_job工程

![image-20220304144712430](/images/posts/2022-3-3-Jenkins/image-20220304144712430.png)



### 3、定时构建（Build periodically） 

> 定时构建可以联想到Linux系统的定时任务
>
> 定时字符串从左往右分别为： 分 时 日 月 周

一些定时表达式的例子：

```
每30分钟构建一次：H代表形参（10:02 --> 10:32）
H/30 * * * * 

每2个小时构建一次: 
H H/2 * * *

每天的8点，12点，22点，一天构建3次： (多个时间点中间用逗号隔开) 
0 8,12,22 * * *

每天中午12点定时构建一次:
H 12 * * *

每天下午18点定时构建一次 
H 18 * * *

在每个小时的前半个小时内的每10分钟 
H(0-29)/10 * * * *

每两小时一次，每个工作日上午9点到下午5点(也许是上午10:38，下午12:38，下午2:38，下午4:38) 
H H(9-16)/2 * * 1-5
```

![image-20220304145035033](/images/posts/2022-3-3-Jenkins/image-20220304145035033.png)

#### 1）pipeline01项目  --- 每隔2分钟构建一次

> */2 * * * *

![image-20220304152325757](/images/posts/2022-3-3-Jenkins/image-20220304152325757.png)



### 4、轮询SCM（Poll SCM）

轮询SCM，是指定时扫描本地代码仓库的代码是否有变更，如果代码有变更就触发项目构建。

![image-20220304145546957](/images/posts/2022-3-3-Jenkins/image-20220304145546957.png)

注意：这次构建触发器，Jenkins会定时扫描本地整个项目的代码，增大系统的开销，不建议使用。



## 2. Git hook自动触发构建

轮询SCM可以实现Gitlab代码更新，项目自动构建，但是 该方案的性能不佳。那有没有更好的方案呢？ 有的。就是利用Gitlab的webhook实现代码push到仓 库，立即触发项目自动构建。

![image-20220304152815251](/images/posts/2022-3-3-Jenkins/image-20220304152815251.png)

### 1、安装Gitlab Hook插件

> 需要安装Generic Webhook Trigger插件，将Jenkins和Gitlab配合起来。

### 2、Jenkins设置自动构建  --- pipeline01项目

> 构建触发器 ----> （Build when a change is pushed to GitLab. GitLab webhook URL: http://192.168.6.20:8888/project/pipeline01）
>
> 最后的URL需要配置到Gitlab中

![image-20220304153845616](/images/posts/2022-3-3-Jenkins/image-20220304153845616.png)

需要把生成的webhook URL配置到Gitlab中。



### 3、Gitlab配置webhook

使用root登陆并修改配置（允许从web钩子和服务向本地网络发出请求）

> Admin Area（Admin） --->  Settings  --->  Outbound requests
>
> 勾选Allow requests to the local network from web hooks and services  ---->  保存

配置Gitlab的项目

> 项目 ---->  setttings  --->   Webhooks   ---> 填选内容 ---->  Add  Webhook
>
> URL粘贴从Jenkins复制过来的地址，（有其他需求的可以选择更多功能）

![image-20220304154856693](/images/posts/2022-3-3-Jenkins/image-20220304154856693.png)

测试

![image-20220304155054326](/images/posts/2022-3-3-Jenkins/image-20220304155054326.png)

测试失败（403状态码：需要Jenkins认证之后才可以）

![image-20220304155203535](/images/posts/2022-3-3-Jenkins/image-20220304155203535.png)

### 4、配置Jenkins对Gitlab上的Webhook认证

> Jenkins  --->  用户    --->   设置   --->   API Token --->  添加新Token   ---->  生成Token （一定要保存）
>
> 11e83324acfe4babec2714d70f4940d7f5

![image-20220304161204335](/images/posts/2022-3-3-Jenkins/image-20220304161204335.png)

重新编辑Gitlab上的Webhook

> 在URL中添加用户和token信息：
>
> http://geray:11e83324acfe4babec2714d70f4940d7f5@192.168.6.20:8888/project/pipeline01

![image-20220304161840037](/images/posts/2022-3-3-Jenkins/image-20220304161840037.png)

测试：

![image-20220304161910388](/images/posts/2022-3-3-Jenkins/image-20220304161910388.png)

成功构建

![image-20220304162106095](/images/posts/2022-3-3-Jenkins/image-20220304162106095.png)



### 5、修改代码并提交，测试

![image-20220304162434677](/images/posts/2022-3-3-Jenkins/image-20220304162434677.png)





## 3. Jenkins参数化构建

有时在项目构建的过程中，我们需要根据用户的输入动态传入一些参数，从而影响整个构建结果，这时 我们可以使用参数化构建。

Jenkins支持非常丰富的参数类型

> 项目  ---->  配置 ----->  General  ---->  描述 ----->  This project is parameterized

![image-20220304170528749](/images/posts/2022-3-3-Jenkins/image-20220304170528749.png)



### 1、Jenkins添加字符串类型参数（按照分支拉取）

![image-20220304171114196](/images/posts/2022-3-3-Jenkins/image-20220304171114196.png)



### 2、修改Jenkinsfile文件（pipeline流水线代码）

![image-20220304171338328](/images/posts/2022-3-3-Jenkins/image-20220304171338328.png)

将代码提交到Gitlab仓库

修改index.jsp页面，添加master分支标记，并提交到Gitlab代码仓库

![image-20220304172451558](/images/posts/2022-3-3-Jenkins/image-20220304172451558.png)

### 3、创建分支

> 项目右键 ----> git ----> repository  --->  branches   ---->  new  branch
>
> 这里创建v1分支

修改index.jsp 页面，添加v1分支标记，并提交到Gitlab代码仓库

![image-20220304172649790](/images/posts/2022-3-3-Jenkins/image-20220304172649790.png)

提交之后的Gitlab仓库中项目存在两个分支

![image-20220304172921868](/images/posts/2022-3-3-Jenkins/image-20220304172921868.png)

### 4、通过参数（这里用不同分支名）进行构建测试

1. 使用master参数值构建master分支提交的信息

![image-20220304173419129](/images/posts/2022-3-3-Jenkins/image-20220304173419129.png)

2. 使用v1参数值构建v1分支提交的信息

![image-20220304173558412](/images/posts/2022-3-3-Jenkins/image-20220304173558412.png)



## 4. 配置邮箱服务器发送构建结果

### 1、安装Email Extension插件

> 邮件发送需要Email Extension Template插件

![image-20220307085423361](/images/posts/2022-3-3-Jenkins/image-20220307085423361.png)



### 2、Jenkins设置邮箱相关参数

> Manage Jenkins   ---->  Configure System 

**设置系统发件人邮箱（这里使用163邮件）：**

> Manage Jenkins   ---->  Configure System  ---->  Jenkins Location

![image-20220307090208243](/images/posts/2022-3-3-Jenkins/image-20220307090208243.png)



**设置邮件参数：**

> Manage Jenkins   ---->  Configure System   ----->   Extended E-mail Notification
>
> 配置的邮箱需要开启STMP协议：（授权码：）GKJBRBCXGFTMWQJP

**获取邮箱授权码：**

![image-20220307094852950](/images/posts/2022-3-3-Jenkins/image-20220307094852950.png)



**邮件通知配置（和上面的Extended E-mail Notification配置基本一致）**

![image-20220307095613641](/images/posts/2022-3-3-Jenkins/image-20220307095613641.png)

测试成功后应用并保存



### 3、准备邮件内容

在项目根目录编写email.html，并把文件推送到Gitlab，内容如下：

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>
</head>
<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
      offset="0">
<table width="95%" cellpadding="0" cellspacing="0"
       style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
    <tr>
        <td>(本邮件是程序自动下发的，请勿回复！)</td>
    </tr>
    <tr>
        <td><h2>
            <font color="#0000FF">构建结果 - ${BUILD_STATUS}</font>
        </h2></td>
    </tr>
    <tr>
        <td><br />
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>项目名称&nbsp;：&nbsp;${PROJECT_NAME}</li>
                <li>构建编号&nbsp;：&nbsp;第${BUILD_NUMBER}次构建</li>
                <li>触发原因：&nbsp;${CAUSE}</li>
                <li>构建日志：&nbsp;<a
                        href="${BUILD_URL}console">${BUILD_URL}console</a></li>
                <li>构建&nbsp;&nbsp;Url&nbsp;：&nbsp;<a
                        href="${BUILD_URL}">${BUILD_URL}</a></li>
                <li>工作目录&nbsp;：&nbsp;<a
                        href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>
                <li>项目&nbsp;&nbsp;Url&nbsp;：&nbsp;<a
                        href="${PROJECT_URL}">${PROJECT_URL}</a></li>
            </ul>
        </td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">Changes Since Last
            Successful Build:</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    编写Jenkinsfile添加构建后发送邮件
    <tr>
        <td>
            <ul>
                <li>历史变更记录 : <a
                        href="${PROJECT_URL}changes">${PROJECT_URL}changes</a></li>
            </ul> ${CHANGES_SINCE_LAST_SUCCESS,reverse=true, format="Changes for
            Build #%n:<br />%c<br />",showPaths=true,changesFormat="<pre>[%a]<br
        />%m</pre>",pathFormat="&nbsp;&nbsp;&nbsp;&nbsp;%p"}
        </td>
    </tr>
    <tr>
        <td><b>Failed Test Results</b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><pre
                style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica,
sans-serif">$FAILED_TESTS</pre>
            <br /></td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">构建日志 (最后 100行):</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><textarea cols="80" rows="30" readonly="readonly"
                      style="font-family: Courier New">${BUILD_LOG,
maxLines=100}</textarea>
        </td>
    </tr>
</table>
</body>
</html>
```

> - ${BUILD_NUMBER} 是Jenkins的变量，表示每次构建的序号
> - ${BUILD_STATUS}  构建状态
>
> 更多具体的参数信息可以到Jenkins查看：Manage Jenkins   ---->  Configure System   ----->   Content Token Reference （点击旁边的问好即可）
>
> ![image-20220307100603823](/images/posts/2022-3-3-Jenkins/image-20220307100603823.png)

![image-20220307100645741](/images/posts/2022-3-3-Jenkins/image-20220307100645741.png)

切换分支到master并提交代码到仓库

![image-20220307143115889](/images/posts/2022-3-3-Jenkins/image-20220307143115889.png)



### 4、编写Jenkinsfile添加构建后邮件发送

将以下内容放到Jenkinsfile的stages之后

```
    post {
        always {
            emailext(
                subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} -  ${BUILD_STATUS}!',
                body: '${FILE,path="email.html"}',
                to: 'xxx@qq.com'
                    )
                }
    }
    
 # subject 邮件标题
 # body  邮件正文
 # to   收件人
```

> - post   构建后操作（所以要放在stages之后）
> - stages   构建操作

![image-20220307141522011](/images/posts/2022-3-3-Jenkins/image-20220307141522011.png)



**post内容也是可以通过Jenkins来进行生成的：**

> pipeline的项目  ---->   配置   --->    流水线配置  ---->  流水线语法   ----->   Declarative Directive  Generator  ---->  post: Post Stage or Build Conditions

将更改后的文件推送到Gitlab，并在Jenkins构建测试：

![image-20220307145403718](/images/posts/2022-3-3-Jenkins/image-20220307145403718.png)

查看控制台输出：

![image-20220307145339231](/images/posts/2022-3-3-Jenkins/image-20220307145339231.png)



*注：我这里测试成功，控制台输出成功，但邮件没有收到信息*

## 5. Jenkins + SonarQube代码审查（1） - 安装SonareQube

### 1、SonaQube简介

![image-20220307162105154](/images/posts/2022-3-3-Jenkins/image-20220307162105154.png)

SonarQube是一个用于管理代码质量的开放平台，可以快速的定位代码中潜在的或者明显的错误。目前 支持java,C#,C/C++,Python,PL/SQL,Cobol,JavaScrip,Groovy等二十几种编程语言的代码质量管理与检 测。 

官网：[https://www.sonarqube.org/](https://www.sonarqube.org/)

### 2、环境要求

| 软件     | 服务         | 版本                         |
| -------- | ------------ | ---------------------------- |
| JDK      | 192.168.6.20 | 1.8                          |
| MySQL    | 192.168.6.20 | 5.7                          |
| sonaQube | 192.168.6.20 | 7.6（高版本的不再支持mysql） |

**安装jdk**

> 下载解压后配置环境变量即可

```
vi /etc/profile

#jdk environment
export JAVA_HOME=/usr/local/jdk1.8.0_291
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
```



**安装mysql**

卸载自带数据库

```
rpm -qa | grep mysql
rpm -qa | grep mariadb
rpm -e --nodeps mariadb-libs-5.5.56-2.el7.x86_64
```

创建用户

```
groupadd mysql
useradd -g mysql mysql -s /sbin/nologin
```

解压并创建配置文件及目录

```
tar xvf  mysql-5.7.36-el7-x86_64.tar.gz -C /usr/local
mv /usr/local/mysql-5.7.36-el7-x86_64 /usr/local/mysql
```

配置文件/etc/my.cnf

```
[client]
port	= 3306
socket	= /tmp/mysql.sock

[mysql]
prompt="\u@db \R:\m:\s [\d]> "
no-auto-rehash

[mysqld]
server-id = 3306100
user	= mysql
port	= 3306
basedir	= /usr/local/mysql
datadir	= /data/mysql/
socket	= /tmp/mysql.sock
pid-file = db.pid

character-set-server = utf8mb4

skip_name_resolve = 1
open_files_limit    = 65535
back_log = 1024
max_connections = 512
max_connect_errors = 1000000
table_open_cache = 1024
table_definition_cache = 1024
table_open_cache_instances = 64
thread_stack = 512K
external-locking = FALSE
max_allowed_packet = 32M
sort_buffer_size = 4M
join_buffer_size = 4M
thread_cache_size = 768

interactive_timeout = 600
wait_timeout = 600
tmp_table_size = 32M
max_heap_table_size = 32M
slow_query_log = 1
slow_query_log_file = /data/mysql/slow.log
log-error = /data/mysql/error.log
long_query_time = 0.1

log-bin = /data/mysql/mysql-binlog
sync_binlog = 1
binlog_cache_size = 4M
max_binlog_cache_size = 1G
max_binlog_size = 1G
gtid_mode = on
enforce_gtid_consistency = 1
log_slave_updates
binlog_format = row
relay_log_recovery = 1
relay-log-purge = 1
key_buffer_size = 32M
read_buffer_size = 8M
read_rnd_buffer_size = 4M
bulk_insert_buffer_size = 64M
#myisam_sort_buffer_size = 128M
#myisam_max_sort_file_size = 10G
#myisam_repair_threads = 1
lock_wait_timeout = 3600
explicit_defaults_for_timestamp = 1
innodb_thread_concurrency = 0
innodb_sync_spin_loops = 100
innodb_spin_wait_delay = 30
master_info_repository = TABLE
relay_log_info_repository = TABLE
slave_parallel_type=LOGICAL_CLOCK

transaction_isolation = REPEATABLE-READ
#innodb_additional_mem_pool_size = 16M
innodb_buffer_pool_size = 1024M
innodb_buffer_pool_instances = 8
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1
innodb_data_file_path = ibdata1:1G:autoextend
innodb_flush_log_at_trx_commit = 1
innodb_log_buffer_size = 32M
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
#innodb_max_undo_log_size = 1G

# 根据您的服务器IOPS能力适当调整
# 一般配普通SSD盘的话，可以调整到 10000 - 20000
# 配置高端PCIe SSD卡的话，则可以调整的更高，比如 50000 - 80000
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
innodb_flush_neighbors = 0
innodb_write_io_threads = 8
innodb_read_io_threads = 8
innodb_purge_threads = 4
innodb_page_cleaners = 4
innodb_open_files = 65535
innodb_max_dirty_pages_pct = 50
innodb_flush_method = O_DIRECT
innodb_lru_scan_depth = 4000
innodb_checksum_algorithm = crc32
#innodb_file_format = Barracuda
#innodb_file_format_max = Barracuda
innodb_lock_wait_timeout = 10
innodb_rollback_on_timeout = 1
innodb_print_all_deadlocks = 1
innodb_file_per_table = 1
innodb_online_alter_log_max_size = 4G
innodb_stats_on_metadata = 0

innodb_status_file = 1
# 注意: 开启 innodb_status_output & innodb_status_output_locks 后, 可能会导致log-error文件增长较快
innodb_status_output = 0
innodb_status_output_locks = 0

#performance_schema
performance_schema = 1
performance_schema_instrument = '%=on'

#innodb monitor
innodb_monitor_enable="module_innodb"
innodb_monitor_enable="module_server"
innodb_monitor_enable="module_dml"
innodb_monitor_enable="module_ddl"
innodb_monitor_enable="module_trx"
innodb_monitor_enable="module_os"
innodb_monitor_enable="module_purge"
innodb_monitor_enable="module_log"
innodb_monitor_enable="module_lock"
innodb_monitor_enable="module_buffer"
innodb_monitor_enable="module_index"
innodb_monitor_enable="module_ibuf_system"
innodb_monitor_enable="module_buffer_page"
innodb_monitor_enable="module_adaptive_hash"

 # Group Replication
#server_id = 1003306
#gtid_mode = ON
#enforce_gtid_consistency = ON
#master_info_repository = TABLE
#relay_log_info_repository = TABLE
binlog_checksum = NONE
#log_slave_updates = ON
#log_bin = binlog
#binlog_format= ROW
transaction_write_set_extraction = XXHASH64

# 未来可能被弃用的变量，会出现告警信息，binlog_expire_logs_seconds用来代替
expire_logs_days = 7
# binlog_expire_logs_seconds = 7

# 8版本弃用的变量
# loose-group_replication_group_name = 'e842862c-9b12-11e8-8131-080027f1fd08'
# internal_tmp_disk_storage_engine = InnoDB # 8版本不再支持，默认引擎
# query_cache_size = 0 # 8版本不再支持这两个参数
# query_cache_type = 0
# loose-group_replication_start_on_boot = off
# loose-group_replication_local_address = 'enmoedu:33066'
# loose-group_replication_group_seeds ='enmoedu1:33067,enmoedu2:33068,enmoedu:33066'
# loose-group_replication_bootstrap_group = off
# loose-group_replication_single_primary_mode=off
# loose-group_replication_enforce_update_everywhere_checks=true

[mysqldump]
quick
max_allowed_packet = 32M
```

创建目录

```
# 创建数据目录和配置文件目录
mkdir -p /data/mysql/
chown mysql:mysql -R /data/mysql
chown mysql:mysql -R /usr/local/mysql/
```

初始化服务

```
cd /usr/local/mysql/bin

./mysqld --defaults-file=/etc/my.cnf --initialize --basedir=/usr/local/mysql --datadir=/data/mysql --user=mysql
```

查看密码

```
more  /data/mysql/error.log 
 100 200 300 400 500 600 700 800 900 1000
 100 200 300 400 500 600 700 800 900 1000 1100 1200 1300 1400 1500 1600 1700 1800 1900 2000
 100 200 300 400 500 600 700 800 900 1000 1100 1200 1300 1400 1500 1600 1700 1800 1900 2000
2022-03-07T11:22:55.405218Z 0 [Warning] InnoDB: New log files created, LSN=45791
2022-03-07T11:22:55.516280Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2022-03-07T11:22:55.581356Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: ef72461f-9e08-11ec-8f9a-000c29c7e54f.
2022-03-07T11:22:55.582090Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2022-03-07T11:22:56.743064Z 0 [Warning] A deprecated TLS version TLSv1 is enabled. Please use TLSv1.2 or higher.
2022-03-07T11:22:56.743081Z 0 [Warning] A deprecated TLS version TLSv1.1 is enabled. Please use TLSv1.2 or higher.
2022-03-07T11:22:56.743878Z 0 [Warning] CA certificate ca.pem is self signed.
2022-03-07T11:22:57.093564Z 1 [Note] A temporary password is generated for root@localhost: gLa-/hpKD3zy
```

添加自动启动

```
# 使用systemctl来管理mysql
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
```

启动数据库并修改密码

```
# 安全启动
./mysqld_safe --defaults-file=/etc/my.cnf &
# 配置mysql环境变量
echo "export PATH=$PATH:/usr/local/mysql/bin" >> /etc/profile
source /etc/profile

mysql -uroot -p
alter user 'root'@'localhost' identified by 'root@123';
flush privileges;

# 设置权限
use mysql;
select user,host from user;
update user set host='%' where user='root';
flush privileges;

# 关闭数据库
./mysqladmin shutdown -uroot -p'root@123'
```



### 3、安装SonarQube

在MySQL创建sonar数据库

```
create database sonar;
```



![image-20220307164627703](/images/posts/2022-3-3-Jenkins/image-20220307164627703.png)

下载sonar压缩包： https://www.sonarqube.org/downloads/

解压sonar，并设置权限

```
# 解压
jar xf sonarqube-7.6.zip
# 创建目录
mkdir /opt/sonar 
# 移动文件
mv sonarqube-7.6/* /opt/sonar/
# 创建sonar用户，必须sonar用于启动，否则报错
useradd sonar 
# 更改sonar目录及文件权限
chown -R sonar. /opt/sonar 

# /etc/sysctl.conf里添加配置
vm.max_map_count=262144
sysctl -p
# 追加/etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536

# 查看修改(重新登陆)
ulimit
```

修改sonar配置文件（注意mysql端口）

- 注意：sonar默认监听9000端口，如果9000端口被占用，需要更改。

```
vi /opt/sonar/conf/sonar.properties
# 内容如下：
sonar.jdbc.username=root 
sonar.jdbc.password=root@123
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false&allowPublicKeyRetrieval=true
```

- mysql8.0版本需要后面添加：allowPublicKeyRetrieval=true

![image-20220307174423064](/images/posts/2022-3-3-Jenkins/image-20220307174423064.png)

启动sonar

```
cd /opt/sonar
# 添加可执行操作
chmod +x ./bin/linux-x86-64/wrapper
chmod +x elasticsearch/bin/elasticsearch
# 启动
su sonar ./bin/linux-x86-64/sonar.sh start 
# 查看状态
su sonar ./bin/linux-x86-64/sonar.sh status 
# 停止
su sonar ./bin/linux-x86-64/sonar.sh stop 
# 查看日志
tail -f logs/sonar.logs 
```

访问：http://192.168.6.20:9000/about

![image-20220307194007319](/images/posts/2022-3-3-Jenkins/image-20220307194007319.png)



默认账户：admin/admin

创建Token（Jenkins整合需要）：

> 点击头像  ---->  My  Account  -->  Security
>
> b14cda9e8c451a5347f57a113891511f858d76b0

![image-20220307195126145](/images/posts/2022-3-3-Jenkins/image-20220307195126145.png)





## 6. Jenkins + SonarQube代码审查（2） -  Jenkins整合sonarqube

![image-20220307195419044](/images/posts/2022-3-3-Jenkins/image-20220307195419044.png)

### 1、安装SonarQube Scanner 插件 

> SonarQube Scanner插件

### 2、添加SonarQube凭证

> Manage Jenkins ---->  Manage Credentials  ---->  全局凭证  ---->  添加凭证

![image-20220308100804302](/images/posts/2022-3-3-Jenkins/image-20220308100804302.png)

### 3、Jenkins配置sonarqube（整合）

> Manage Jenkins ---->  Global Tool Configuration   --->   SonarQube Scanner  ---->   新增SonarQube Scanner

![image-20220308084902464](/images/posts/2022-3-3-Jenkins/image-20220308084902464.png)

应用、保存

> Manage Jenkins --->   Configure System  ---->   SonarQube servers  ---->  Add SonarQube

![image-20220308101011184](/images/posts/2022-3-3-Jenkins/image-20220308101011184.png)





## 7. Jenkins + SonarQube代码审查（3） -  非流水线项目

### 1）自由风格（web_demo_freestyle）

> 项目名  --->  配置  ---->  构建  --->  添加构建步骤   --->  Execute SonarQube Scanner
>
> 将一下内容添加到Analysis properties
>
> 或者使用Path to project properties指定项目中的文件

```
# must be unique in a given SonarQube instance
sonar.projectKey=web_demo_freestyle
# this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
sonar.projectName=web_demo_freestyle
sonar.projectVersion=1.0

# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set.
# 需要扫描的内容 （exclusions排除那些）
sonar.sources=.
sonar.exclusions=**/test/**,**/target/**
sonar.java.source=1.8
sonar.java.target=1.8

# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```

![image-20220308143410443](/images/posts/2022-3-3-Jenkins/image-20220308143410443.png)

应用、保存并构建

刷新sonarqube页面

> 可以看到一些代码问题：
>
> Bugs：一些重大的安全隐患
>
> Code Smells：可以优化的地方
>
> Duplications：重复的代码片段

![image-20220308144328706](/images/posts/2022-3-3-Jenkins/image-20220308144328706.png)



#### 模拟错误代码进行测试

> 并在pom.xml中添加所需的依赖包（pom外添加-导入会在jenkins构建时找不到文件）

```
package com.geray;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.doPost(req,resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    //模拟错误代码
        int i = 100/0;

        //模拟冗余代码
        int a = 100;
        a = 200;

        resp.getWriter().write("Hello Servlet!");
    }
}
```



![image-20220308154419222](/images/posts/2022-3-3-Jenkins/image-20220308154419222.png)

提交到仓库并重新构建查看

![image-20220308174828548](/images/posts/2022-3-3-Jenkins/image-20220308174828548.png)





## 8. Jenkins + SonarQube代码审查（4） -  流水线项目 

### 1）项目根目录下，创建sonar-project.properties文件

> 将Analysis properties中的内容移动到项目根目录下的sonar-project.properties文件中
>
> 注意项目名称

![image-20220308174651059](/images/posts/2022-3-3-Jenkins/image-20220308174651059.png)

### 2）修改Jenkinsfile，加入SonarQube代码审查阶段

> 将一下内容放置到stages中，可以在项目构建之前或者项目部署之前进行检查

```
        stage('SonarQube代码审查') {
            steps{
                script {
                    //引入sonarqubeScanner工具
                    scannerHome = tool 'sonarqube-scanner'
                }
                withSonarQubeEnv('sonarqube6.7.4') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
```

成功被构建

![image-20220308173102758](/images/posts/2022-3-3-Jenkins/image-20220308173102758.png)

