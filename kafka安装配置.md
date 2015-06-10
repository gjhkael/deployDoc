# kafka安装配置

kafak的安装配置分为两部分：zookeeper的安装配置和kafaka的安装配置\

###zookeeper安装配置
   ***简介：*** zookeeper是一个为分布式应用提供一致性服务的软件，它是开源的Hadoop项目的一个子项目，并根据google发表的一篇论文来实现的。zookeeper为分布式系统提供了高笑且易于使用的协同服务，它可以为分布式应用提供相当多的服务，诸如统一命名服务，配置管理，状态同步和组服务等。zookeeper接口简单，我们不必过多地纠结在分布式系统编程难于处理的同步和一致性问题上，你可以使用zookeeper提供的现成(off-the-shelf)服务来实现来实现分布式系统额配置管理，组管理，Leader选举等功能。

zookeeper集群的安装,准备三台服务器server1:192.168.8.198,server2:192.168.8.199,server3:192.168.8.220(这是我工作中测试用的环境，可以根据自身集群机器ip自行调整)
 - [下载]zookeeper（本人使用3.4.6版本，选个最新版本即可）到自己设置好的想要安装的目录下（我用的/home/havstack/Projects/）
 - 解压压缩包： tar -zxvf zookeeper-3.4.6.tar.gz
 - 配置zookeeper,首先将conf/zoo_sample.cfg拷贝一份命名为zoo.cfg (先到zookeeper目录下然后使用命令：cp conf/zoo_sample.cfg zoo.cfg) 配置如下：

    ```
    # The number of milliseconds of each tick
    tickTime=2000
    # The number of ticks that the initial 
    # synchronization phase can take
    initLimit=10
    # The number of ticks that can pass between 
    # sending a request and getting an acknowledgement
    syncLimit=5
    # the directory where the snapshot is stored.
    # do not use /tmp for storage, /tmp here is just 
    # example sakes.
    dataDir=/home/havstack/Projects/zookeeper-3.4.6/data
    dataLogDir=/home/havstack/Projects/zookeeper-3.4.6/log
    # the port at which the clients will connect
    clientPort=2181
    # the maximum number of client connections.
    # increase this if you need to handle more clients
    #maxClientCnxns=60
    #
    # Be sure to read the maintenance section of the 
    # administrator guide before turning on autopurge.
    #
    # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
    #
    # The number of snapshots to retain in dataDir
    #autopurge.snapRetainCount=3
    # Purge task interval in hours
    # Set to "0" to disable auto purge feature
    #autopurge.purgeInterval=1
    server.1=192.168.8.198:3888:4888
    server.2=192.168.8.199:3888:4888
    server.3=192.168.8.220:3888:4888
    ```
 -配置说明 
  
    tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
    dataDir：顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。
    
    clientPort：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
    
    initLimit：这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 5个心跳的时间（也就是 tickTime）长度后 Zookeeper服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
    
   syncLimit：这个配置项标识 Leader 与Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是2*2000=4 秒
   
    server.A=B：C：D：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号
    
    注意:dataDir,dataLogDir中的havsatck是当前登录用户名，data，log目录开始是不存在，需要使用mkdir命令创建相应的目录。并且分别在data和log目录下创建文件myid,serve1,server2,server3该文件内容分别为1,2,3。
针对服务器server2,server3可以将server1复制到相应的目录，不过需要注意dataDir,dataLogDir目录,并且文件myid内容分别为2,3

***启动zookeeper***
到zookeeper目录下：bin/zkServer.sh start

出现类似以下内容:
    >JMX enabled by default
    >Using config: /home/wwb/zookeeper/bin/../conf/zoo.cfg
    >Starting zookeeper ... STARTED

bin/zkCli.sh -server192.168.8.199:2181

出现类似以下内容:

   INFO  [main-SendThread(192.168.8.199:2181):ClientCnxn$SendThread@1235] - Session establishment complete on server 192.168.8.199/192.168.8.199:2181, sessionid = 0x24ddb2c43490005, negotiated timeout = 30000
   
###kafka安装配置###
 - [下载kafka0.8]保存到服务器相应的目录下（我用的/home/havstack/Projects/），这里可以下载源码或则直接下载编译好的kafka程序。这里分别介绍。

    1）先介绍简单的直接下载编译好的程序然后配置
    
    - 解压安装包：tar -zxvf kafka_2.10-0.8.2.1.tgz 然后mv kafka_2.10-0.8.2.1 kafka
    - 修改kafka/config/server.properties配置属性
    
    ```
    broker.id=1   //第一个服务器为1，第二个的话就为2.以此类推
     port=9092 
     host.name=192.168.8.198
     num.network.threads=3  
     num.io.threads=8  
     socket.send.buffer.bytes=1048576  
    socket.receive.buffer.bytes=1048576  
     socket.request.max.bytes=104857600  
    log.dir=./logs  
    num.partitions=2  
    log.flush.interval.messages=10000  
    log.flush.interval.ms=1000  
    log.retention.hours=168  
    #log.retention.bytes=1073741824  
    log.segment.bytes=536870912  
    num.replica.fetchers=2  
    log.cleanup.interval.mins=10  
    zookeeper.connect=192.168.0.1:2181,192.168.0.2:2182,192.168.0.3:2183  
    zookeeper.connection.timeout.ms=1000000  
    ```

  2）[下载源码]进行编译
    > ./sbt update  
    > ./sbt package  
    > ./sbt assembly-package-dependency
  
  然后在按上面进行配置。
  
***启动***

分别在各个服务器上启动kafka使用下面命令：
 bin/kafka-server-start.sh config/server.properties &
 
 ***测试***
 
Create a topic
> 1)bin/kafka-topics.sh --create --zookeeper 192.168.8.198:2181 --replication-factor 3 --partitions 1 --topic test

> 2)bin/kafka-topics.sh --list --zookeeper 192.168.8.198:2181

Send some messages
> bin/kafka-console-producer.sh --broker-list 192.168.8.198:9092 --topic test

    This is a message
    This is another message
Start a consumer
> bin/kafka-console-consumer.sh --zookeeper 192.168.8.198:2181 --topic test --from-beginning 打印如下信息：

    This is a message
    This is another message







[下载]:http://zookeeper.apache.org/releases.html
[下载kafka0.8]:https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.1/kafka_2.10-0.8.2.1.tgz
[下载源码]:https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.1/kafka-0.8.2.1-src.tgz