---
layout: post
title: "maven和IDEA工具的的安装、配置和项目构建"
description: "分享"
tag: maven
---

# 1、卸载IDEA

> 进入IDEA安装的bin目录双击卸载文件进行卸载

卸载之后要彻底清理IDEA的残留文件

## 1. 清理注册表

win + R 快捷键，输入`regedit` 打开注册表

> 1. 点击一级菜单 `HKEY_CURRENT_USER`， 右键查找，输入idea，会找到jetbrains，然后，右键删除。

![image-20220302183226057](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302183226057.png)



> 2. 再来一次，点击一级菜单 HKEY_CURRENT_USER， 右键查找，输入jetbrain，会找到jetbrain相关，然后，右键删除。

![image-20220302183405423](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302183405423.png)

## 2. 清理卸载残留文件

主要存在以下地方：（最好彻底删除）

```
C:\user\${用户名称}\ideaProjects\
C:\Users\${用户名称}\AppData\Roaming\JetBrains
C:\Users\Public\.jetbrains

C:\Program Files\JetBrains
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\JetBrains\
```

之后重启系统即可



# 2、安装IDEA
### 1.安装

双击安装文件`ideaIU-2020.1.exe`

1. 选择路径

![image-20220302184129452](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302184129452.png)

2. 安装配置

![image-20220302184239347](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302184239347.png)

3. 一路默认即可，最后直接运行

![image-20220302184445376](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302184445376.png)


### 2.激活

运行之后，先选择免费试用

![image-20220302184604614](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302184604614.png)

准备好破解工具，并放到特定位置（建议破解之后破解工具原路径保留，不要删除）

![image-20220302184826537](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302184826537.png)

将破解文件`jetbrains-agent-latest.zip`拖入以下窗口中进行破解

![image-20220302185007148](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302185007148.png)



点击`Restart`重启，应用更改

![image-20220302185130144](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302185130144.png)

点击安装

![image-20220302185154662](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302185154662.png)

安装成功并重启IDEA





# 3、IDEA配置Maven环境

## 1. 下载安装和环境变量配置

下载Maven：[https://archive.apache.org/dist/maven/maven-3/](https://archive.apache.org/dist/maven/maven-3/)

1. 根据自己的操作系统进行下载

![image-20220302185642427](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302185642427.png)

2. 解压到指定路径（比如我的：`D:\MyWork`）

3. 需要配置所依赖的`JAVA_HOME` 和 `MAVEN_HOME`

![image-20220302191505461](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302191505461.png)



4. 验证

![image-20220302191653341](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302191653341.png)



## 2. 仓库配置

### 自定义仓库位置

1. 创建仓库目录`D:\MyWork\repository`

2. 修改maven的`settings.conf`文件

```
# 默认位置：Default: ${user.home}/.m2/repository
<localRepository>/path/to/local/repo</localRepository>
 
# 自定义仓库位置
<localRepository>D:\MyWork\repository</localRepository>
```

### 镜像仓库配置

#### 1、查看默认的连接仓库位置

> 以下操作忽略位置！
>
> maven的安装目录下的lib目录中随便找一个jar文件
>
> 右击使用解压工具（WinRAR）打开，回到上层目录，使用查找搜索所有以pom开头的文件（`pom*.*`）

![image-20220303100538182](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303100538182.png)

在搜索结果总找到一个名为`pom-4.0.0.xml`，点击定位，切到改文件的具体位置

![image-20220303101148884](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303101148884.png)

把改为件提取出来，并打开文件，查看默认的仓库连接位置

![image-20220303101746835](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303101746835.png)

```
  <repositories>
    <repository>
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>
```

#### 2、配置国内阿里云镜像仓库

> 打开settings.xml，找到镜像配置位置粘贴以下内容

```
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

![image-20220303104108952](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303104108952.png)

#### 3、全局setting和用户setting的区别

- 全局settting定义了当前计算器中Maven的公共配置
- 用户settting定义了当前用户的配置

> 用户setting可以将maven的全局setting复制到repository并修改自己的镜像仓库即可

## 3. IDEA配置Maven

### 1、创建一个空工程



![image-20220302190203568](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302190203568.png)



填写项目名称和工作路径：

![image-20220302190132495](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302190132495.png)

完成

### 2、 配置JDK

1. 修改Project的SDK为我们自己的jdk

![image-20220302190346775](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220302190346775.png)

### 3、配置Maven

> File ---> Settings ---> Build Tools ---> Maven
>
> 注意：修改Maven的目录之后，仓库位置会自动变更（没有变更的可能是版本不匹配或者setting配置不合适导致的）

![image-20220303104303426](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303104303426.png)

至此ok！



# 4、构建Maven项目

快捷键：Ctrl + Alt + Shift + s 创建一个工程

## 1. 第一个Maven项目（不使用模板）

### 创建

> Project Settings  ---> Modules ---> New Module ----> Maven

![image-20220303105926343](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303105926343.png)

填写项目名称和项目唯一标识

![image-20220303110048352](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303110048352.png)

下一步，都是灰色的目录（不是一个标准的maven结构）；对目录进行标记

![image-20220303110250291](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303110250291.png)

标记之后还需要添加一个目录`resources`，并做相关标记（删除旁边的添加信息可以恢复标记）

![image-20220303110510811](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303110510811.png)



也可以在项目结构中右击目录，选择`Mark Directory as`进行更改

![image-20220303111045974](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303111045974.png)

maven的管理（生命周期和插件）

![image-20220303111631242](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303111631242.png)

### 添加测试jar包

> 一定要刷新，并生成相关的信息（Dependencies）
>
> 这里版本可以设置到4.13.2

![image-20220303112100037](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303112100037.png)

### 创建类

```
    public String say(String name) {
        System.out.println("Hello "+name);
        return "hello " + name;
    }
```



![image-20220303112455592](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303112455592.png)

创建类

![image-20220303112630430](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303112630430.png)



![image-20220303112922862](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303112922862.png)

### 创建测试类

```
    @Test
    public void testSay(){
        demo demo = new demo();
        String ret = demo.say("geray");
        // 添加断言
        Assert.assertEquals("hello geray",ret);
    }
```



![image-20220303113257616](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303113257616.png)

至此表示maven工程就绪

### 构建

> maven生米周期  ---->  双击clean  ---->  双击complie

![image-20220303113653757](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303113653757.png)



### 测试

> 双击test
>
> 测试类需要注解@Test

![image-20220303121541032](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303121541032.png)



### 自定义maven的构建命令

> 方便后期的debug断点调试

![image-20220303140031770](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303140031770.png)

![image-20220303140242605](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303140242605.png)

完成之后便可以进行相关操作

![image-20220303140315163](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303140315163.png)



## 2. 第二个Maven项目（使用模板快速自动构建）

> Project Settings  ---> Modules ---> New Module ----> Maven
>
> 使用模板中的`maven-archetype-quickstart`

![image-20220303140811037](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303140811037.png)

填写信息

![image-20220303140951886](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303140951886.png)

下一步

![image-20220303141014062](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303141014062.png)

项目结构：（如果相关信息没有被标记或者缺少某些目录可以进行如下管理）

![image-20220303141302341](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303141302341.png)

添加配置如下

![image-20220303141533142](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303141533142.png)



## 3. 第三个Maven项目（web项目）

> Project Settings  ---> Modules ---> New Module ----> Maven
>
> 使用模板中的`maven-archetype-quickstart`

### 1、创建

![image-20220303141949809](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303141949809.png)

配置

![image-20220303142030370](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303142030370.png)

> 构建之后项目结构的src下缺少test相关信息，和main下缺少java
>
> 添加相关目录并标记

![image-20220303142456384](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303142456384.png)

添加一个jsp测试页面

![image-20220303142728209](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303142728209.png)

页面内容

![image-20220303142833648](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303142833648.png)



### 2、Tomcat插件安装和web项目启动

#### 清理web.xml内容（默认的很多有错误）

清理为如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4"
         xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">


</web-app>
```



![image-20220303143126484](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303143126484.png)



#### 清理pom.xml文件并添加tomcat插件 

清理如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

  <modelVersion>4.0.0</modelVersion>
  <packaging>war</packaging>

  <name>web01</name>
  <groupId>com.geray</groupId>
  <artifactId>web01</artifactId>
  <version>1.0-SNAPSHOT</version>

  <dependencies>
  </dependencies>

</project>

```



![image-20220303143326108](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303143326108.png)

maven的坐标库中搜索`tomcat maven`

> 地址：[https://mvnrepository.com/](https://mvnrepository.com/)

![image-20220303143641274](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303143641274.png)

使用较为稳定的版本

![image-20220303143740096](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303143740096.png)

复制信息到pod.xml中

![image-20220303144154160](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303144154160.png)

粘贴到pom.xml中，并刷新等待ok后，双击`tomcat7:run`，启动tomcat服务

![image-20220303144510803](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303144510803.png)



复制地址到浏览器访问：

![image-20220303144800522](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303144800522.png)

修改Tomcat端口和访问地址

> 修改pom.xml，在tomcat插件信息添加如下信息

```
          <groupId>org.apache.tomcat.maven</groupId>
          <artifactId>tomcat7-maven-plugin</artifactId>
          <version>2.1</version>
          <configuration>
            <port>80</port>
            <path>/</path>
          </configuration>
```

![image-20220303145136408](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303145136408.png)

启动访问测试

![image-20220303145218064](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303145218064.png)



自定义启动

![image-20220303145443543](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303145443543.png)

应用

![image-20220303145532996](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303145532996.png)

直接点击按钮运行

![image-20220303145607548](/images/posts/2022-3-2-IDEA与Maven（安装、配置、项目构建）/image-20220303145607548.png)
