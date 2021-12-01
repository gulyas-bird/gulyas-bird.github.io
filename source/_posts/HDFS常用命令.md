---
title: HDFS常用命令
date: 2019-03-31 16:58:41
tags: hadoop
---
### 1 配置用户环境变量
gulyas是hadoop的专属用户，hadoop的目录所属用户及所属用户组都是gulyas。将hadoop的命令配置在gulyas用户的用户环境变量中，只能由gulyas用户才能操作。在配置环境变量的时候，.bashrc和.bash_profile两个文件都可以配置环境变量，但是应该首选.bashrc文件，因为在ssh的时候，会主动刷新用户的.bashrc文件。从而保证我们的环境变量配置可以生效。
也可以在/etc/profile.d/目录下将自定义的环境变量封装到xxx.sh文件中。linux同样会扫描.sh并加载到PATH环境变量中。
<!-- more -->
```shell
su - gulyas
vi .bashrc
export HADOOP_HOEM=/home/gulyas/app/hadoop
export PATH=$HADOOP/bin:$HADOOP/sbin:$PATH

source .bashrc
which hadoop # 检查是否配置成功
```
[](./1.png)
### 2 目录访问协议
```text
scheme://authority/path
```
> scheme表示协议
> authority表示权限

例如：HDFS文件系统的协议名：hdfs，本地文件系统的协议为：file。
### 3 hdfs dfs -ls
```shell
hdfs dfs -ls       # 列出工作主目录下的信息
hdfs dfs -ls /     # 列出hdfs文件系统中
```

### 4 hdfs dfs -mkdir
```shell
# path可以是绝对路径，也可以是相对路径。
hdfs dfs -mkdir [-p] <path>

hdfs dfs -mkdir tmp   # 在hdfs文件系统中/user/gulyas目录下创建tmp目录
hdfs dfs -mkdir ./tmp # 同上 hdfs dfs -ls 查看

hdfs dfs -mkdir /tmp  # 在hdfs文件系统的根目录下创建一个tmp目录
hdfs dfs -ls /        # 查看根路径下的文件列表
```

### 5 hdfs dfs -rm -rf
```shell
hdfs dfs -rm [-r] [-f] <uri>  # 删除目录或文件，-r -f不能组合成-rf
hdfs dfs -rm -r -f /test      # 删除根目录下的test目录
hdfs dfs -rmdir /test         # 删除目录：只能删除空目录
```
### 6 hdfs dfs -appendToFile
```shell
# appendToFile命令既可以将一个或多个文件添加到HDFS中，也可以将流中的数据读取到HDFS中。最终都是在hdfs中生成一个文件。
hdfs dfs -appendToFile <localSrc> <dst>

#将本地文件exp.log上传到hdfs中/user/gulyas/tmp目录，并重命名为：exception.log
hdfs dfs -appendToFile ./exp.log ./tmp/exception.log

#将本地的test目录传到hdfs中，重命名为tst文件【注意这里并不是目录】。
hdfs dfs -appendToFile ./test ./tst
```
### 7 hdfs dfs -cat
```shell
hdfs dfs -cat <URI>
# 查看/user/gulyas/tmp/exception.log 文件内容
hdfs dfs -cat ./tmp/exception.log
```

### 8 hdfs dfs -find
```shell
# 从根目录下精确搜索exception.log文件
hdfs dfs -find / -name exception.log
# 从/user/gulyas目录下搜索名称中包含ex字符串的文件
hdfs dfs -find /user/gulyas -name '*ex*'
```

### 9 hdfs dfs -put
从本地文件系统拷贝文件到hdfs中。
```shell
hdfs dfs -put [-f] [-p] [-l] [-d] [-t <thread count>] [ - | <localsrc1> .. ]. <dst>
# -f 如果已存在就覆盖
# -p 递归拷贝

hdfs dfs -put head.png tmp/head.png # 拷贝文件
hdfs dfs -put txt/ tmp/txt          # 将目录txt拷贝到hdfs中的/user/gulyas/tmp/txt
```
### 10 hdfs dfs -get
从hdfs中下载文件到本地文件系统中。
```shell
hdfs dfs -get [-ignorecrc] [-crc] [-p] [-f] <src> <localdst>
# -p 保留访问权限 修改时间等信息
# -f 如果目标文件已存在，直接覆盖。
hdfs dfs -get ./tmp ./hdfs-temp-dic # 将hdfs中的tmp目录下载到本地并重命名
```

### 11 hdfs dfs -cp
```shell
hdfs dfs -cp [-f] [-p | -p[topax]] URI [URI ...] <dest>
# -f 如果存在，直接覆盖。

hdfs dfs -cp /user/hadoop/file1 /user/hadoop/file2
hdfs dfs -cp /user/hadoop/file1 /user/hadoop/file2 /user/hadoop/dir

hdfs dfs -cp tmp ./temp # 将tmp拷贝并重命名为temp
```

### 12 hdfs dfs -count
统计目录下文件夹数量 文件数量 目录下文件总字节数。
```shell
hadoop fs -count [-q] [-h] [-v] [-x] [-t [<storage type>]] [-u] [-e] [-s] <paths>

hdfs dfs -count  /user/gulyas  # 对/user/gulyas目录进行统计
```
结果每列含义：目录数 文件数 总大小（字节） 目录名称

### 13 hdfs dfs -mv
```shell
hdfs dfs -mv URI [URI ...] <dest>
# mv命令只能在hdfs文件系统中使用，不能跨系统。

hdfs dfs -mv tmp /tmp_home
```

### 14 hdfs dfs -chown
```shell
hdfs dfs -chmod [-R] <MODE[,MODE]... | OCTALMODE> URI [URI ...]
# -R 递归目录授权
```
### 15 hdfs dfs -chown
```shell
hdfs dfs -chown [-R] <MODE[,MODE]... | OCTALMODE> URI [URI ...]

hdfs dfs -chown gulyas:gulyas temp # 更文件改用户组
hdfs dfs -chmod 700 temp # 给temp目录授权700
```

### 16 hdfs dfs -tail
```shell
hadoop fs -tail [-f] URI # 输出文件的末尾输出到控制台

# -f 动态输出
```

### 17 hdfs dfs -touch
```shell
hdfs dfs -touch [-a] [-m] [-t TIMESTAMP] [-c] URI [URI ...]
```




### 18 hdfs dfs -touchz
```shell
hdfs dfs -touchz URI [URI ...] # 创建一个长度为0的文件
```


