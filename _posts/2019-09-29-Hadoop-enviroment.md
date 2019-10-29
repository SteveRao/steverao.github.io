---
layout:     post
title:      "大数据系列（二）：Hadoop集群搭建"
subtitle:   "Hadoop集群搭建"
date:       2019-09-29
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 开发工具
      大数据
      Hadoop
---





## 总体概述

> 这是一篇迟到了一年的博文，去年学习Spark SQL 期间曾为跑分布式任务搭建过该集群一次，由于当时对很多东西都不熟悉花了不少时间。这次为了大数据实验又要重新搭建一次环境，但相对于第一次来说快多了。另外非常不建议在单机中搭建集群来完成课程实验之类的任务，因为涉及到集群间的调用等等问题速度很慢，[Hadoop的本地模式](https://blog.csdn.net/l_15156024189/article/details/81810553)也许是更好的选择！*（注意：本文搭建集群所使用的操作系统是Ubuntu 16.04）*
>



## 1.安装配置jdk

- 首先进入[官网](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)下载对应的jdk安装包，进入官网下载页面，选择接受协议，点击对应的版本即可开始下载（下载需要登录Oracle账号，没有的注册一下）

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/dl-jdk.png" width="70%"/><b>jdk下载</b></div>

- 下载后，将压缩包复制到ubuntu**当前用户steve**的主目录下，执行如下命令将其移到待安装位置：

  ```Shell
  # 移动
  steve@master:~$ sudo mv jdk-8u231-linux-x64.tar.gz /usr/local 
  # 解压
  steve@master:~$ sudo tar -zxvf jdk-8u231-linux-x64.tar.gz
  # 重命名
  steve@master:~$ sudo mv jdk1.8.0_231 jdk
  ```


- 配置java环境变量，在当前用户目录下，执行`vim ~/.bashrc`

  ```Shell
  export JAVA_HOME=/usr/local/jdk
  export JRE_HOME=${JAVA_HOME}/jre
  export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
  export PATH=${JAVA_HOME}/bin:$PATH
  ```

  当然也可以通过`sudo vim /etc/profile`设置系统环境变量，但并不推荐这么做。修改成功以后执行如下命令使配置文件修改立即生效。

  ```Shell
  steve@master:~$ source ~/.bashrc
  ```

- 以上步骤完成后，输入`java -version`命令，结果如下则表示成功：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/jdk-success.png" width="70%"/><b>jdk配置成功</b></div>

  ​



## 2.下载安装Hadoop

- jdk安装后，接下来进行Hadoop的下载安装，首先进入[官网](https://hadoop.apache.org/releases.html)选择所需版本的hadoop安装包。本文所选安装版本如下图所示：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/dl-hadoop.png" width="70%"/><b>hadoop下载</b></div>

  下载后，与jdk安装过程类似。首先，将压缩包拷贝到linux操作系统当前用户主目录下。然后，在分别执行如下命令，完成文件的移动，解压和重命名以及权限修改等等。

  ```Shell
  # 移动
  steve@master:~$ sudo mv hadoop-2.8.5.tar.gz /usr/local 
  # 解压
  steve@master:~$ sudo tar -zxvf hadoop-2.8.5.tar.gz
  # 重命名
  steve@master:~$ sudo mv hadoop-2.5.8 hadoop
  # 修改权限，由于/usr/local目录中的文件或文件夹权限属于root用户，所以需要修改成steve用户所有，不然后# 续集群搭建过程中会有权限问题
  steve@master:~$ sudo chown -R steve:steve /usr/local/hadoop
  ```

- 解压后，接下来开始配置hadoop对应的环境变量。与jdk环境变量配置类似，执行如下命令并在打开的文件中添加如下内容：

  ```Shell
  steve@master:~$ vim ~/.bashrc
  # 末尾添加内容如下两行
  export HADOOP_HOME=/usr/hadoop
  export PATH=${HADOOP_HOME}/bin:$PATH
  #修改后，执行source ~/.bashrc让修改立即生效
  ```

  完成后，在终端执行hadoop version显示结果类似下图则表示安装成功：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/hadoop-success.png" width="70%"/><b>hadoop安装验证</b></div>

- 在完成了以上最基本的jdk和hadoop安装以及环境变量配置后。接下来，就可以利用VMWare虚拟机的克隆功能克隆出master主机的两个副本，我们可以分别为其命名worker1和worker2，克隆后的虚拟机拥有与原主机一模一样的系统和文件。在这一步进行克隆免去了重复安装基本软件的工作。接下来我们要分别为三台虚拟机配置ip与域名映射，执行如下命令并在打开的文件中添加如下内容：

  ```Shell
  steve@master:~$ sudo vim /etc/hosts
  # 末尾添加如下内容，这里的ip地址与需要与自己当前节点具体ip地址对应，通过ifconfig命令查看本机ip
  192.168.10.138 master
  192.168.10.141 worker1
  192.168.10.142 worker2
  ```

  三台主机hosts文件配置好后，再分别为刚克隆出来的worker节点重命名，操作命令如下：

  ```Shell
  steve@master:~$ sudo hostname [名称]
  #我们这名称分别就是worker1和worker2
  ```

  上述过程后完成后，我们接下来就开始完成一些集群中节点相关的配置。

  ​



## 3.使用SSH实现集群通信

为了实现集群之间任务调度过程中互相传输数据的通畅性，一般采用**Secure Shell**(安全外壳协议,SSH)完成主机间的认证与数据传输。通过步骤二目前我们已经有三台主机了：master, worker1和worker2。配置集群间的SSH通信配置过程如下：*（注意：下文主要是在worker2节点上演示）*

- 首先，分别在每一台主机上安装SSH server，安装命令如下：

  ```Shell
  steve@worker2:~$ sudo apt-get install openssh-server
  ```

- 然后，在每一个主机上生成公私钥，生成命令如下：

  ```Shell
  steve@worker2:~$ ssh-keygen -t rsa
  ```

  命令执行过程中，**所有的输入请求直接输入回车即可**，成功以后的显示如下图所示：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/ssh-keygen.png" width="70%"/><b>ssh公私钥生成</b></div>

- 秘钥生成后，执行如下命令将本机公钥添加进认证文件末尾：

  ```Shell
  steve@worker2:~$ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
  ```

  完成命令后，我们可以在本地上验证一下公钥添加是否成功。执行如下命令登录worker2，第一次登录会要求输入密码，公钥添加成功以后后续登录就可免密。第一次登录页面结果如下图所示：

  ```Shell
  # 首先使用如下命令关闭集群中所有节点的防火墙，不然无法登录
  steve@worker2:~$ sudo ufw disable
  # ssh 远程免密登录
  steve@worker2:~$ ssh worker2
  ```

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/ssh-auth.png" width="70%"/><b>ssh单机验证</b></div>

- 分别在三台主机上完成以上步骤，每台主机便可实现在本机上的免密登录。但对于集群来说这样没有任何意义，需要实现主机之间的免密登录才可保证集群的通信顺畅。因此需要将本机的公钥信息添加到集群中其他主机的认证文件末尾。该过程需要使用**scp**文件传输协议，具体操作过程如下：

- 首先，通过以下命令将worker节点上的公钥文件传输给master节点，执行命令与成功结果分别如下：

  ```Shell
  steve@worker2:~$ scp .ssh/id_rsa.pub  steve@master: /home/steve/.ssh/id_rsa2.pub
  #id_rsa2.pub是在master主机上该文件的名字，主要是为了区别不同的worker节点上的公钥文件
  ```

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/scp-pub.png" width="70%"/><b>传输公钥</b></div>

  当master节点拥有了所有worker节点的公钥后，再通过如下命令将所有的公钥添加进master节点的authorized_keys文件末尾：

  ```Shell
  steve@master:~$ cat .ssh/id_rsa[x].pub >> .ssh/authorized_keys
  #括号[x]的分别表示分别表示1和2
  ```

  将所有的公钥添加到认证文件中以后，现在还只有master节点集群所有节点的公钥信息。因此还需要将master节点修改后的认证文件分别传输给集群中的所有worker节点，这个过程我们还是使用scp协议来实现：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/scp-auth.png" width="70%"/><b>传输认证文件</b></div>

- 当集群中所有节点的认证文件包括了其他节点的公钥以后，就可以互相免密登录了。具体测试可以用**ssh +节点域名**方式。




## 4.配置Hadoop集群

#### Hadoop集群配置

- **配置core-site.xml** 

  修改Hadoop核心配置文件`/usr/local/hadoop/etc/hadoop/core-site.xml`，通过`fs.default.name`指定NameNode的IP地址和端口号，通过`hadoop.tmp.dir`指定hadoop数据存储的临时文件夹。 

  ```xml
  <configuration>
      <property>
          <name>hadoop.tmp.dir</name>
          <value>file:/usr/local/hadoop/tmp</value>
          <description>Abase for other temporary directories.</description>
      </property>
      <property>
          <name>fs.defaultFS</name>
          <value>hdfs://master:9000</value>
      </property>
  </configuration>
  ```

- **配置hdfs-site.xml** 

  修改HDFS核心配置文件`/usr/local/hadoop/etc/hadoop/hdfs-site.xml`，通过`dfs.replication`指定HDFS的备份因子为3，通过`dfs.name.dir`指定namenode节点的文件存储目录，通过`dfs.data.dir`指定datanode节点的文件存储目录。 

  ```xml
  <configuration>
      <property>
          <name>dfs.replication</name>
          <value>3</value>
      </property>
      <property>
          <name>dfs.name.dir</name>
          <value>/usr/local/hadoop/hdfs/name</value>
      </property>
      <property>
          <name>dfs.data.dir</name>
          <value>/usr/local/hadoop/hdfs/data</value>
      </property>
  </configuration>
  ```

- **配置mapred-site.xml** 

  拷贝mapred-site.xml.template为mapred-site.xml，在进行修改 

  ```Shell
  cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml  
  vim /usr/local/hadoop/etc/hadoop/mapred-site.xml
  ```

  ```xml
  <configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
     <property>
        <name>mapred.job.tracker</name>
        <value>http://master:9001</value>
    </property>
  </configuration>
  ```

- **配置yarn-site.xml** 

  ```xml
  <configuration>
  <!-- Site specific YARN configuration properties -->
      <property>
          <name>yarn.nodemanager.aux-services</name>
          <value>mapreduce_shuffle</value>
      </property>
      <property>
          <name>yarn.resourcemanager.hostname</name>
          <value>master</value>
      </property>
  </configuration>
  ```

- **配置masters文件** 

  修改`/usr/local/hadoop/etc/hadoop/masters`文件，该文件指定namenode节点所在的服务器机器。删除localhost，添加namenode节点的主机名master；不建议使用IP地址，因为IP地址可能会变化，但是主机名一般不变。 

  ```Shell
  vim /usr/local/hadoop/etc/hadoop/masters
  ## 内容
  master
  ```

- 配置slaves文件*（master主机特有）* 

  修改`/usr/local/hadoop/etc/hadoop/slaves`文件，该文件指定哪些服务器节点是datanode节点。删除locahost，添加所有datanode节点的主机名，如下所示。 

  ```Shell
  vim /usr/local/hadoop/etc/hadoop/slaves
  ## 内容
  worker1
  worker2
  master
  ```


- **hadoop配置jdk环境变量**

  修改hadoop-env.sh文件，在其中添加之前设置过的jdk路径

  ```Shell
  # 根据实际情况设置
  export JAVA_HOME=/usr/local/jdk
  ```

- **worker节点配置hadoop环境**

  这里我们还是通过scp协议将master节点配置好的hadoop目录下的相关文件传输给各个worker节点替换其原来的hadoop目录。

  ```Shell
  # worker[x]根据实际情况修改
  steve@master:~$ scp -r /usr/local/hadoop steve@worker[x]:/usr/local/hadoop
  ```

  不过，在将hadoop目录传给了所有的worker节点后，worker节点需要将hadoop目录中的slave文件删除，因为该文件只有master节点才拥有。

#### Hadoop集群启动

- **格式化HDFS文件系统** 

  进入master的~/hadoop目录，执行以下操作 

  ```Shell
  steve@master:hadoop$ bin/hadoop namenode -format
  ```

  **格式化namenode，第一次启动服务前执行的操作，以后不需要执行。** *（注意：多次格式化可能会出现[datanode节点无法启动](https://blog.csdn.net/baidu_16757561/article/details/53698746)问题）*

- **在master节点启动hadoop集群**

  ```Shell
  steve@master:hadoop$ sbin/start-all.sh
  ```

  集群启动成功后，使用jps工具查看节点中的进程，master和worker节点的结果分别如下图则表示成功：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/hadoop-master-process.png" width="50%"/><b>hadoop集群master节点中进程</b></div>

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/hadoop-worker-process.png" width="50%"/><b>hadoop集群worker节点中进程</b></div>

- 最后，还可以在浏览器中通过192.168.10.138:50070*（master ip地址:端口）*的方式访问hadoop集群的可视化管理界面，如下图所示：

  <div align="center"><img src="/img/in-post/content/bigdata/hadoop/hadoop-ui.png" width="70%"/><b>hadoop集群管理UI</b></div>




## 参考资料

- [Hadoop分布式集群搭建](http://www.ityouknow.com/hadoop/2017/07/24/hadoop-cluster-setup.html)
- [Hadoop本地模式、伪分布式和全分布式集群安装与部署](https://blog.csdn.net/l_15156024189/article/details/81810553)