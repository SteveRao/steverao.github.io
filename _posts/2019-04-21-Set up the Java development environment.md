---
layout:     post
title:      "搭建Java开发环境"
subtitle:   "配置Java EE开发环境"
date:       2019-04-21
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: JAVA开发
      配置环境
---





### 总体概述
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2019/04/21/Set-up-the-Java-development-environment/)
>
> java EE开发基础环境主要包括java开发工具**Jdk**，jar包管理工具**Maven**以及程序运行服务器**Tomcat**。虽然网上有很多相关环境搭建的博客或者资料，但每次需要搭建时找起来还是挺费时间，所以决定写一篇博客详细记录相关工具搭建过程为以后提供便利。                                                                                                                                             



### 1.软件安装
---
##### 1.1 Jdk安装

&emsp;&emsp;首先到[Oracle官网](https://www.oracle.com/technetwork/java/javaee/downloads/jdk8-downloads-2133151.html)下载需要的Jdk版本，本文演示的是在windows环境下安装Jdk1.8版本示例。

![](/img/in-post/content/java-development-enviroment/jdk-download.bmp)

&emsp;&emsp;下载需要登录Oracle账号(可免费注册)，成功之后，点击对应的`jdk-XXX-windows-XXX.exe`可执行文件进行安装即可。



##### 1.2 Maven安装

&emsp;&emsp;首先到[Maven官网](https://maven.apache.org/download.cgi)下载对应的安装版本，如下图所示：

![](/img/in-post/content/java-development-enviroment/maven-download.bmp)

&emsp;&emsp;windows环境中下载图示`apache-maven-XXX-bin.zip`的可执行版本即可，右下角可找到其他安装版本。其中`Source zip archive`是Maven的未编译源代码版本，其一般多作为参考学习使用。另外，.tar.gz和.zip主要是两种压缩格式，windows环境下载.zip版本，mac等Linux环境下载.tar.gz版本。



##### 1.3 Tomcat安装

&emsp;&emsp;首先到官网下载需要的安装版本，详见下图：

![](/img/in-post/content/java-development-enviroment/tomcat-download.bmp)

&emsp;&emsp;如上图所示，Tomcat有两种安装包形式，第一种为免安装版本，像Maven一样解压后即可使用。第二种是.exe类型的安装包，下载后像Jdk一样根据提示一步步安装即可，这种方式对新手更省事。



### 2.配置环境变量
---
&emsp;&emsp;“环境变量”一般是指在操作系统中用来指定操作系统运行环境的一些参数，如一些软件或工具的安装位置等。Java开发环境变量主要涉及以下三种：

- **XXX_HOME环境变量：**例如JAVA_HOME其指向Jdk的安装目录，Eclipse/IDEA/Tomcat等软件就是通过搜索JAVA_HOME变量来找到并使用安装好的Jdk。

- **PATH环境变量：**作用是指定命令搜索路径，在shell下执行命令时，它会到PATH变量所指定的路径中查找看是否能找到相应的命令程序。我们需要把 jdk安装目录下的bin目录增加到现有的PATH变量中，bin目录中包含经常要用到的可执行文件如javac、java和javadoc等。设置好 PATH变量后，就可以在任何目录下执行javac和java等工具了。 

- **CLASSPATH环境变量：**作用是指定类搜索路径，要使用已经编写好的类，前提当然是能够找到它们了，JVM就是通过CLASSPATH来寻找类的。我们需要把jdk安装目录下的lib子目录中的dt.jar和tools.jar设置到CLASSPATH中。当然，当前目录“**.**”也必须加入到该变量中。



##### 2.1 jdk环境变量配置

**JAVA_HOME：**`C:\Program Files (x86)\Java\jdk1.8.0_91`（根据个人路径修改）

**PATH：**`%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin;`

**CLASSPATH:**  `.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;`

&emsp;&emsp;配置完成后验证如下则表示成功：

![](/img/in-post/content/java-development-enviroment/jdk-validation.bmp)



##### 2.2 Maven环境变量配置

**MAVEN_HOME:**  `D:\Software\apache-maven-3.5.4`

**PATH:**  `%MAVEN_HOME%\bin;`

&emsp;&emsp;配置完成后验证如下则表示成功：

![](/img/in-post/content/java-development-enviroment/maven-validation.bmp)



##### 2.3 Tomcat环境变量

&emsp;&emsp;正常使用Tomcat部署应用不需要配置环境变量。

&emsp;&emsp;通过双击Tomcat包bin目录下的**startup.bat**启动即后，在浏览器中数据127.0.0.1:8080得到如下页面则表示成功：

![](/img/in-post/content/java-development-enviroment/tomcat-validation.bmp)



### 3 总结
---
&emsp;&emsp;本文主要对Java EE运行基础环境搭建进行了介绍，其中配置环境变量除了可以通过图形界面方式配置以外。在一些企业的开发机中如果不能编辑图像界面还可通过运行**配置脚本**或者在管理员权限下使用**setx**命令进行配置等。



### 参考资料
---
- [Tomcat安装博客](http://www.jeecms.com/hjdj/479.htm)
- [菜鸟教程配置jdk环境变量](http://www.runoob.com/java/java-environment-setup.html)

