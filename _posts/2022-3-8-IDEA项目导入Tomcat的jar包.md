---
layout: post
title: "IDEA工具中继承HttpServlet报错解决"
description: "分享"
tag: IDEA
---

# 1、IDEA工具中继承HttpServlet报错解决

![image-20220308150607450](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308150607450.png)

这是由于缺少web的httpServlet相关jar文件导致；IDEA中导入tomcat的httpservlet相关的jar包

# 2、配置Tomcat服务（没有解决）

## 1. 安装Tomcat服务

[https://tomcat.apache.org/](https://tomcat.apache.org/)

配置环境变量：

> CATALINA_HOME=路径
>
> PATH中添加如下：
>
> %CATALINA_HOME%\bin
>
> %CATALINA_HOME%\lib
>
> CLASS_PATH中添加如下：
>
> %CATALINA_HOME%\lib\servlet-api.jar;

验证：

> win  + r 打开命令窗口输入命令startup

![image-20220308152009249](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308152009249.png)

## 2. IDEA配置Tomcat

> run  --->   Edit Configurations   ---->   点击+好  ，选择Tomcat Server 以及 Local  
>
> 选择Tomcat的安装位置进行配置

![image-20220308152301963](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308152301963.png)

# 3、IDEA导入Tomcat的httpServlet的jar包（可解决，Jenkins中构建会存在遗留问题）

> Ctrl + Shift + Alt + s 快速打开项目配置
>
> Project settings  ---->   Libraries  点击+好，选择java

![image-20220308152842977](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308152842977.png)

## 遗留问题：Jenkins使用maven构建时找不到javax.servlet包

![image-20220308161615640](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308161615640.png)



# 4、解决方案

> pom.xml中添加依赖
>
> ```
>   <dependencies>
>     <dependency>
>       <groupId>org.zenframework.z8.dependencies.servlet</groupId>
>       <artifactId>servlet-api-2.5</artifactId>
>       <version>2.0</version>
>     </dependency>
>   </dependencies>
> ```

> 输入<artifactId>的值，根据提示选择所需的jar文件并刷新maven项目即可

![image-20220308174434726](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308174434726.png)



Jenkins构建结果：

![image-20220308174520154](/images/posts/2022-3-8-IDEA项目导入Tomcat的jar包/image-20220308174520154.png)

