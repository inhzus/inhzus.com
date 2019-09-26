---
title: Hadoop 2.7.1 ECS 单机伪分布配置
date: 2019-03-31 15:10:41
tags: [Hadoop, Configuration, Linux]
---

Hadoop 2.7.1 ECS 单机伪分布配置

### 目的

按照网上某些配置, Jobhistory Server 不能正确显示任务, 或 Resource Manager 8088 无法访问.

综合官方文档及 ML Wiki 配置作小修改后, 配置成功. 参考文档见文末.

### 步骤

```bash
cd ${HADOOP_HOME}/etc/hadoop
```

方便编辑该目录下的各个配置文件.

#### 环境变量

##### /etc/profile

```bash
export JAVA_HOME=/usr/lib/jvm/jdk***
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export HADOOP_HOME=/home/hadoop/hadoop-2.7.1
export PATH=${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:$PATH
```

可将 `${HADOOP_HOME}/bin` `${HADOOP_HOME}/sbin` 添加至 PATH.

##### hadoop-env.sh

```bash
export JAVA_HOME=/usr/lib/jvm/jdk***
```

配置Hadoop 中的 JAVA 环境.

<!--more-->

#### 配置文件

首先配置为单机伪分布

##### core-site.xml

```xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://localhost:9000</value>
        </property>
</configuration>
```

##### hdfs-site.xml

```xml
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
</configuration>
```

之后配置 Jobhistory Server

##### mapred-site.xml

```bash
cp mapred-site.xml.example mapred-site.xml
```

```xml
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>
```

##### yarn-site.xml

```xml
<configuration>
        <property>
                <name>yarn.nodemanager.aux-services</name> 
                <value>mapreduce_shuffle</value>
        </property>
</configuration>                                           
```

#### SSH 无密码配置

检查无密码 ssh 登录

```bash
ssh localhost
```

否则

```bash
ssh-keygen localhost
# 全部回车默认
ssh-copy-id localhost
```

#### 格式化节点

```bash
hadoop namenode -format
```

### 启动

不推荐 `start-all.sh`, 使用

#### 守护进程

```bash
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver
```

##### 检查

```bash
hadoop@server:~/hadoop-2.7.1/etc/hadoop$ jps
5413 JobHistoryServer
4455 DataNode
5703 ResourceManager
4649 SecondaryNameNode
5835 NodeManager
4300 NameNode
10284 Jps
```

#### 创建用户文件夹

```bash
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/hadoop
```

### 运行

```bash
hdfs dfs -put your-file input
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount input output1
hdfs dfs -get output1 ~/output1
```

此时查看 your-ip:19888 即可看到历史记录.

### 参考文档

[ML Wiki: Hadoop Pseudo Distributed Mode](http://mlwiki.org/index.php/Hadoop_Pseudo_Distributed_Mode)

[Apache Hadoop 2.7.1: Setting up a Single Node Cluster](https://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-common/SingleCluster.html)

