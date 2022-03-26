---
layout: post
title: 'Kafka集群部署'
subtitle: 'Kafka集群部署'
date: 2022-03-16
categories: mq
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: mq kafka
---

# Kafka集群部署

## 1. 系统环境准备（所有节点）

| 节点 | IP |
| --- | --- |
| node1 | 192.168.244.128 |
| node2 | 192.168.244.129 |
| node3 | 192.168.244.130 |
1. 关闭防火墙，selinux，设置hostname
2. 安装JDK8以上的版本
    - 从下面的地址下载jdk8
      
        [https://www.oracle.com/java/technologies/downloads/#java8](https://www.oracle.com/java/technologies/downloads/#java8)
        
    - rpm安装jdk
      
        ```bash
        [root@node1 share]# rpm -ivh jdk-8u321-linux-x64.rpm 
        warning: jdk-8u321-linux-x64.rpm: Header V3 RSA/SHA256 Signature, key ID ec551f03: NOKEY
        Preparing...                          ################################# [100%]
        Updating / installing...
           1:jdk1.8-2000:1.8.0_321-fcs        ################################# [100%]
        Unpacking JAR files...
        	tools.jar...
        	plugin.jar...
        	javaws.jar...
        	deploy.jar...
        	rt.jar...
        	jsse.jar...
        	charsets.jar...
        	localedata.jar...
        ```
        
    - 添加环境变量
      
        ```bash
        vi ~/.bash_profile
        
        ## 追加以下内容
        export JAVA_HOME=/usr/java/jdk1.8.0_321-amd64/
        export PATH=$JAVA_HOME/bin:$PATH
        
        ## 环境变量生效
        source ~/.bash_profile
        
        ```
        
    - 验证Java安装是否完成
      
        ```bash
        [root@node1 bin]# java -version
        java version "1.8.0_321"
        Java(TM) SE Runtime Environment (build 1.8.0_321-b07)
        Java HotSpot(TM) 64-Bit Server VM (build 25.321-b07, mixed mode)2.下载
        ```
        

## 2. Zookeeper安装（所有节点）

1. 下载ZooKeeper
   
    ```bash
    wget --no-check-certificate https://dlcdn.apache.org/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz
    ```
    
2. 解压到 /usr/zookeeper目录
   
    ```bash
    mkdir /usr/zookeeper/
    tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz -C /usr/zookeeper/
    ```
    
3. 拷贝生成 zoo.cfg 配置文件
   
    ```bash
    # 进入配置目录
    cd /usr/zookeeper/apache-zookeeper-3.6.3-bin/conf
    
    # 生成 zoo.cfg
    cp zoo_sample.cfg zoo.cfg
    ```
    
4. 编辑zoo.cfg 启动配置
   
    ```bash
    vi zoo.cfg
    
    # 修改下面的值
    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/data/zookeeper
    clientPort=2181
    server.1=node1:2888:3888
    server.2=node2:2888:3888
    server.3=node3:2888:3888
    ```
    
    <aside>
    💡 参数说明
    **tickTime**：指 zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔。
    **initLimit**：用来指定zookeeper 集群中leader接受follower初始化连接时最长能忍受的心跳时间间隔数。若超过 指定个心跳的时间间隔leader还没有收到follower的响应，表明follower连接失败。
    **syncLimit**：指定 leader与follower之间请求和应答时间最长不能超过多少个 tickTime 
    **dataDir**：快照日志的存储路径。
    **dataLogDir**：事物日志的存储路径，如果不配置该路径，当zk吞吐量较大的时，会严重影响zk的性能。
    **clientPort**：客户端连接 zookeeper 服务器的端口，默认为2181。
    **server.1** 这个1是服务器的标识也可以是其他的数字，**这个标识要写到dataDir目录下面myid文件里。**
    server.1=hostname:2888:3888，hostname后面的第一个端口2888主要用于leader和follower之间的通信，第二个端口3888主要用于选主
    
    </aside>
    
5. 创建myid文件(三个节点不一致)
   
    myid文件位置必须放在zoo.cfg配置的dataDir路径下，且文件对应的内容与配置文件中**server.X**中**X**的值保持一致，即node1中myid内的值为1，node2中myid内容为2
    
    ```bash
    vi /data/zookeeper/myid
    
    1
    ```
    
6. 启动zookeeper
   
    ```bash
    [root@node1 bin]# cd /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin
    [root@node1 bin]# ./zkServer.sh start
    /usr/bin/java
    ZooKeeper JMX enabled by default
    Using config: /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    ```
    
7. 查看是否部署成功以及节点状态
   
    ```bash
    [root@node1 bin]# ./zkServer.sh status
    ZooKeeper JMX enabled by default
    Using config: /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
    Client port found: 2181. Client address: localhost. Client SSL: false.
    Mode: follower
    ```
    
8. 设置开机自启动
    - 进入/etc/init.d目录，新建zookeeper脚本
      
        ```bash
        cd /etc/init.d
        
        vi zookeeper
        ```
        
        填充以下内容
        
        ```bash
        #!/bin/bash    
        #chkconfig:2345 20 90    
        #description:zookeeper    
        #processname:zookeeper    
        #自己的Java目录 以及 zookeeper目录
        export JAVA_HOME=/usr/java/jdk1.8.0_321-amd64/
        case $1 in    
                start) su root /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh start;;    
                stop) su root /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh stop;;    
                status) su root /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh status;;    
                restart) su root /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh restart;;    
                *) echo "require start|stop|status|restart" ;;    
        esac
        ```
        
    - 给脚本添加权限
      
        ```bash
        chmod +x zookeeper
        ```
        
    - 尝试通过service启停zookeeper
      
        ```bash
        [root@node1 init.d]# service zookeeper start
        ZooKeeper JMX enabled by default
        Using config: /usr/zookeeper/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
        Starting zookeeper ... already running as process 11197.
        ```
        
    - 添加到开机自启
      
        ```bash
        # 添加开机自启
        chkconfig --add zookeeper
        # 查看是否开启自启
        chkconfig --list zookeeper
        ```
        

## 3. 部署Kafka集群（各个节点）

1. 下载Kafka二进制包
   
    ```bash
    wget https://archive.apache.org/dist/kafka/2.0.0/kafka_2.11-2.0.0.tgz
    ```
    
2. 解压至/usr/kafka目录
   
    ```bash
    mkdir /usr/kafka
    tar -zxvf kafka_2.11-2.0.0.tgz -C /usr/kafka
    ```
    
3. 添加环境变量
   
    ```bash
    vi ~/.bash_profile
    ##追加以下内容
    export KAFKA_HOME=/usr/kafka/kafka_2.11-2.0.0
    export PATH=$PATH:$KAFKA_HOME/bin
    
    ## 生效环境变量
    source ~/.bash_profile
    ```
    
4. 修改server配置
   
    ```bash
    cd /usr/kafka/kafka_2.11-2.0.0/config
    vi server.properties
    ```
    
    | 参数名称 | 参数值 | 备注 |
    | --- | --- | --- |
    | broker.id | 0 | broker.id的值三个节点要配置不同的值，分别配置为0，1，2 |
    | log.dirs | /usr/log/kafka | Kafka日志数据目录 |
    | num.partitions | 1 | 分区数，根据自行修改 |
    | log.retention.hours | 24 | 日志保存时间 |
    | zookeeper.connect | node1:2181,node2:2181,node3:2181 | zookeeper连接地址，多个以逗号隔开 |
    - node1
    
    ```bash
    broker.id=0
    log.dirs=/usr/log/kafka
    log.retention.hours=24
    zookeeper.connect=node1:2181,node2:2181,node3:2181
    ```
    
    - node2
    
    ```bash
    broker.id=1
    log.dirs=/usr/log/kafka
    log.retention.hours=24
    zookeeper.connect=node1:2181,node2:2181,node3:2181
    ```
    
    - node3
    
    ```bash
    broker.id=2
    log.dirs=/usr/log/kafka
    log.retention.hours=24
    zookeeper.connect=node1:2181,node2:2181,node3:2181
    
    ```
    
5. 启动Kafka
   
    ```bash
    cd /usr/kafka/kafka_2.11-2.0.0/bin
    kafka-server-start.sh ../config/server.properties
    
    #查看是否启动
    [root@node1 bin]# jps
    1057 QuorumPeerMain
    12467 Kafka
    12916 Jps
    
    ```
    
6. 创建topic
   
    ```bash
    [root@node2 bin]# ./kafka-topics.sh --create --zookeeper node1:2181,node2:2181,node3:2181 -partitions 3 --replication-factor 3 --topic streaming
    Created topic "streaming".
    ```
    
7. 查看topic
   
    ```bash
    [root@node2 bin]# ./kafka-topics.sh --list --zookeeper node1:2181
    streaming
    ```