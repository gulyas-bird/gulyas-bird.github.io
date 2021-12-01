---
title: Hadoop环境搭建
date: 2019-03-29 14:35:00
tags: hadoop
---
## 1 安装jdk
路径：usr/java/jdk
```shell
mkdir /usr/java # 路径：usr/java/jdk
// tar mv ...

# 配置环境变量
vi /etc/profile 
export JAVA_HOME=/usr/java/jdk
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=$JAVA_HOME/lib:$CLASSPATH

# 刷新环境变量
source /etc/profile

# 验证
java -version  #或者which java
```
<!-- more -->
## 2 服务器准备
### 2.1 下载上传hadoop
1、去hadoop.apache.org下载hadoop，使用版本：3.2.2
2、上传hadoop-3.2.2.tar.gz到服务器/tmp下。
说明：/tmp目录会定时清除没有使用的文件，默认30天。
### 2.2 新建用户、创建工作目录
```shell
useradd gulyas
su - gulyas # 不配置密码
# 系统/tmp目录有定时自动清除策略，在用户目录建一个自己用
mkdir sourcecode software app log data lib tmp
```
### 2.3 移动解压
```shell
su -
mv /tmp/hadoop-3.2.2.tar.gz /home/gulyas/software/
chown gulyas:gulyas /home/gulyas/software/hadoop-3.2.2.tar.gz
su - gulyas
# -C 解压到指定目录
tar -zxvf /home/gulyas/hadoop-3.2.2.tar.gz -C /home/gulyas/app/
```
### 2.4 建立软连接
```shell
ln -s hadoop-3.2.2 hadoop   # 这里是相对路径，使用绝对路径也行
```
hadoop目录说明
```shell
bin      # hadoop相关命令
etc      # 配置文件
include
lib      # 存放Hadoop的本地库（对数据进行压缩解压缩功能）
libexec
LICENSE.txt
NOTICE.txt
README.txt
sbin    # hadoop服务启动停止脚本
share   # 存放Hadoop的依赖jar包、文档、和官方案例
```
### 2.5 hadoop配置jdk
```shell
su - gulyas
vi ./app/hadoop/etc/hadoop/hadoop-env.sh
# 加入以下配置
export JAVA_HOME=/usr/java/jdk
```
![](./1.png)
### 2.6 配置主机名
```shell
hostnamectl set-hostname yogie.com
ifconfig # 查看内网ip
vi /etc/hosts
192.168.0.119 yogie.com  # 一定要是本机内网ip

# hosts中不能配置公网ip，否则可能导致9000端口程序访问不到。
# 重启也会导致NameNode起不来。
# 但是如果要配置其他主机NameNode的地址，那一定要配置其他主机的公网ip。

vi /etc/sysconfig/network
HOSTNAME=yogie.com
```
## 3 配置伪分布式模式
hadoop的配置文件都在`HADOOP_HOME/etc`目录下：
### 3.1 core-site.xml
```xml
<configuration>
    <!--配置NameNode的启动端点-->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://yogie.com:9000</value>
    </property>
    <!--配置hadoop的数据目录-->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/gulyas/tmp/hadoop-${user.name}</value>
    </property>
</configuration>
```
说明：`hadoop.tmp.dir`在没有配置的情况下已经启动过，如果直接改配置文件的此项配置，会导致NameNode服务启动失败。那是因为，hadoop的每个进程每次启动都会生成一个版本文件，NameNode的版本文件位置：
```shell
${hadoop.tmp.dir}/hadoop-gulyas/dfs/name/current/VERSION
```
VERSION文件内容如下：
[](./2.png)
`${hadoop.tmp.dir}`默认位置在/tmp。在修改配置文件前，停掉所有的hadoop服务。然后将`/tmp/hadoop-gulyas`目录拷贝到`/home/gulyas/tmp/`下重新启动即可。
```shell
mv /tmp/hadoop-gulyas /home/gulyas/tmp/
```
### 3.2 workers
workers中配置的是DataNode的端点，将原来locahost改为主机名。
```text
yogie.com
```
### 3.3 hdfs-site.xml
```xml
<configuration>
    <!--配置block副本数量，默认为3-->
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <!--配置Secondary NameNode的启动端点(http协议)-->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>yogie.com:9868</value>
    </property>
    <!--配置Secondary NameNode的启动端点(https协议)-->
    <property>
        <name>dfs.namenode.secondary.https-address</name>
        <value>yogie.com:9869</value>
    </property>
    <!--如果一台机器挂载了多个数据盘，那么需要做一下配置：
        <property>
            <name>dfs.datanode.data.dir</name>
            <value>/data01/dfs/dn,/data02/dfs/dn,/data03/dfs/dn</value>
        </property>
    -->
</configuration>
```
说明：如果一台机器挂载了多块物理磁盘，需要对`dfs.datanode.data.dir`做配置。
例如：一块磁盘的写能力30M/s，装载10快磁盘后，就是300M/s，写同样的数据，后者更高效。多块磁盘是为了存储空间更大，且高效率的读写IO。 肯定比单块磁盘更快。所以在生产上，DataNode的`dfs.datanode.data.dir`参数必须根据机器的实际情况配置。
### 3.4 hadoop-env.sh
pid文件说明：
pid文件记录集群中每个进程启动的pid编号。当执行sbin/stop-dfs.sh或stop-all.sh等命令的时候，hadoop会根据pid文件找到每个进程的pid，然后执行kill -9 pid来关闭进程。
```shell
export HADOOP_PID_DIR=/home/gulays/tmp
```
配置pid文件的位置：
[](./3.png)
### 3.5 mapred-site.xml
```xml
<configuration>
    <!--配置mr作业的计算框架-->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <!--配置mr运行application的classpath目录-->
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```
### 3.6 yarn-site.xml
```xml
<configuration>
    <!--NodeManager上运行的附属服务-->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!--环境变量白名单-->
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <!--yarn作业web界面，yarn的8088端口不安全，必须改端口-->
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>yogie.com:8123</value>
    </property>
</configuration>
```
## 4、配置ssh
目的：使得【ssh gulyas.com】能够免密远程登录。
```shell
ssh-keygen # 一直回车
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 测试
ssh yogie.com # 第一次需要输入yes
# 如果还需要输入密码，那么ssh配置或者600权限有问题。
```
[](./4.png)
## 5、启动
### 5.1 启动hdfs
1. 格式化hdfs文件目录
```shell
cd app/hadoop
# 执行格式化命令
bin/hdfs namenode -format
```
2. 启动主节点和数据节点
> NameNode：存储的是数据的元数据，例如文件名称，路径，大小等信息。
> DataNode：存储的是数据。
```shell
sbin/start-dfs.sh
# 启动成功之后，使用jsp查看一下
# 或者使用ps查看
ps -ef|grep hadoop
```
根据控制台输出的启动日志：
> 1. hadoop在yogie.com主机启动了一个NameNode。
> 2. 同时启动了一个数据节点：DataNode
> 3. 最后启动了一个SecondaryNameNode节点（作为NodeNode的副节点）。
[](./5.png)
3. hadoop的日志目录
```shell
/hadoop/logs
```
4. 访问web界面（在云配置开放端口）
```shell
http://公网ip:9870/
```
### 5.2 启动yarn
1. 启动进程
```shell
sbin/start-yarn.sh
jps # 查看进程是否启动
```
[](./6.png)
2. 访问web界面
```shell
http://公网ip:8123
```
[](./7.png)
## 6 HDFS官方案例测试
### 6.1 运行案例
```shell
# 在hdfs中创建一个目录
bin/hdfs dfs -mkdir /user
# 在/user目录下创建一个gulyas目录，相当于创建了一个gulyas的用户目录
# gulyas对应的是当前光标的用户
bin/hdfs dfs -mkdir /user/gulyas

# 新建目录，注意：这里的文件夹并没有加相对路径或者是绝对路径
# 其含义为：在hdfs中、当前用户目录下创建一个input目录
bin/hdfs dfs -mkdir input

bin/hdfs dfs -ls /uer/gulyas # 查看用户目录
Found 1 items
drwxr-xr-x  - gulyas supergroup 0 2021-11-24 23:15 /user/gulyas/input

# 拷贝文件：从当前物理主机中/home/gulyas/app/hadoop/etc/hadoop/*.xml传到hdfs文件系统中
bin/hdfs dfs -put etc/hadoop/*.xml input

# 运行
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar grep input output 'dfs[a-z.]+'

# 查看hdfs文件系统中，自动生成的output文件目录下的内容
bin/hdfs dfs -ls output
```
### 6.2 上传与下载
这里说的上传与下载指的是本地物理机文件系统与hdfs文件系统进行交互。
1. 下载：例如将gulyas用户目录中的output文件目录下载到本地文件系统中。
```shell
bin/hdfs dfs -get output ./test/output

# 将output文件下载到本地文件系统中的tmp目录中
/home/gulyas/app/hadoop/bin/hdfs dfs -get output /home/gulyas/tmp/output
```
2. 上传：例如本文中3.2节所示，将本地文件系统的input目录下的文件上传到hdfs文件系统中。
```shell
bin/hdfs dfs -put etc/hadoop/*.xml input
```
3. 查看文件内容
```shell
bin/hdfs dfs -cat output/*
```
## 7. yarn官方案例测试
运行一个官方提供的案例包hadoop-mapreduce-examples-3.2.2.jar中的wordcount方法。wordcount的功能就是统计一个文本文件中的英文单词个数。
```shell
# 上传一个测试文件
hdfs dfs -put /wc/input/wc_in.txt

# 执行测试命令
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar wordcount /wc/input/wc_in.txt /wc/output
```
案例命令说明：
```text
yarn jar      #运行一个jar包
share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar  #案例的jar包
wordcount     #案例中函数的参数
/wc/input/wc_in.txt    #输入参数
/wc/output             #输出参数，这个文件目录必须是一个不存在的目录
```
```shell
# 查看结果
hdfs dfs -ls /wc/output
hdfs dfs -cat /wc/output/part-r-00000
```
案例源码地址：github地址：https://github.com/apache/hadoop/blob/trunk/hadoop-mapreduce-project/hadoop-mapreduce-examples/src/main/java/org/apache/hadoop/examples/WordCount.java


