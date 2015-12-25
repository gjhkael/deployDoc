# ubuntu环境下配置ganglia监控

Ganglia是UC Berkeley发起的一个开源实时监视项目，用于测量数以千计的节点，为云计算系统提供系统静态数据以及重要的性能度量数据。Ganglia系统基本包含以下三大部分。
- **Gmond**：Gmond运行在每台计算机上，它主要监控每台机器上收集和发送度量数据（如处理器速度、内存使用量等）。
 - **Gmetad**：Gmetad运行在Cluster的一台主机上，作为Web Server，或者用于与Web Server进行沟通。
 - **Ganglia Web前端**：Web前端用于显示Ganglia的Metrics图表。

### 1、首先在Ubuntu安装LAMP服务。
 - 1）安装Apache  
```
    sudo apt-get install apache2  //在浏览器中输入localhost查看是否成功
```
 - 2）安装php5
```
    sudo apt-get install php5 libapache2-mod-php5 php5-mysql
```
 - 3）安装mysql 
```
    sudo apt-get install mysql-server mysql-client //过程中输入root用户密码即可 
    sudo mysql -uroot -p                           //验证数据库是否安装成功
```
 - 4）安装phpmyadmin （可选）
```
    sudo apt-get install libapache2-mod-auth-mysql phpmyadmin
    其中会弹出一些选择菜单，按默认的就ok
    cp /etc/phpmyadmin/apache.conf /etc/apache2/sites-available/phpmyadmin
    cd /etc/apache2/sites-enabled/
    sudo ln -s ../sites-available/phpmyadmin
    sudo /etc/init.d/apache2 restart   //在浏览器中输入localhost/phpmyadmin验证是否配置成功
```
### 2、安装Ganglia
```
    sudo apt-get install rrdtool ganglia*
```

### 3、配置Ganglia
 - 1）**主节点配置**
```
    cd /etc/ganglia/
    sudo vim gmond.conf
```
将cluster配置下信息进行更改
```
cluster {
  name = "unspecified"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
```
改为如下所示：
```
cluster {
  name = "sparkganglia"  //名字自己随意取，最总要在所有配置文件中一致
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
```
```
udp_send_channel {
  mcast_join = 239.2.11.71
  port = 8649
  ttl = 1
}
```
改为
```
  #mcast_join = 239.2.11.71
  host = 192.168.8.49  //master的ip地址
  port = 8649
  ttl = 1
```
```
udp_recv_channel {
  mcast_join = 239.2.11.71
  port = 8649
  bind = 239.2.11.71
}
```
改为
```
udp_recv_channel {
 # mcast_join = 239.2.11.71
  port = 8649
 # bind = 239.2.11.71
}
```
```
    sudo vim gmetad.conf
```
```
data_source "my cluster" localhost
```
改为
```
data_source "sparkganglia" localhost
```
在master节点上通过下面方法重启服务。
```
    sudo /etc/init.d/ganglia-monitor start
    sudo /etc/init.d/gmetad start
    sudo /etc/init.d/apache2 restart
    在浏览器中输入：localhost/ganglia 查看主节点的监控状态。
```

 - 2）**从节点配置**

从节点只需要安装ganglia-monitor即可
```
    sudo apt-get install ganglia-monitor
```
```
    sudo vim /etc/ganglia/gmond.conf
```
将
```
cluster {
  name = "my cluster"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
```
改为
```
cluster {
  name = "sparkganglia"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
```
```
udp_send_channel {
  mcast_join = 239.2.11.71
  port = 8649
  ttl = 1
}
```
改为
```
  #mcast_join = 239.2.11.71
  host = 192.168.8.49  //master的ip地址
  port = 8649
  ttl = 1
```
```
udp_recv_channel {
  mcast_join = 239.2.11.71
  port = 8649
  bind = 239.2.11.71
}
```
改为
```
udp_recv_channel {
 # mcast_join = 239.2.11.71
 # port = 8649
 # bind = 239.2.11.71
}
```
最后执行
```
    sudo /etc/init.d/ganglia-monitor restart
```
在刷新监控页面，查看资源是不是加入到监控中。
### 4、监控集群
```
    在浏览器中输入：localhost/ganglia 
```
如下图所示：![图](https://github.com/gjhkael/deployDoc/blob/master/image/ganglia.png)





