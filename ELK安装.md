# Centos 7.x 下安装elk 6.4版本

## 环境介绍
- 64位 centos

- java环境  
java 的安装请参见java环境配置相关文档。
	```
	java -version
	echo $JAVA_HOME
	```
##这次安装3个组件：  
	elasticsearch  
	kibana  
	filebeat或logstash  
    elastic接收日志等信息，可以通过filebeat 也可以用logstash，这两个都可以发送日志到elasticsearch，如果只是简单的日志监控，这两个服务单独都可以完成日志上报。

## 用yum安装elk

### yum源
- 导入源key：
	rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
- 创建yum 源配置文件

```
[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

### 安装elasticsearch
```
sudo yum install elasticsearch
sudo dnf install elasticsearch
sudo zypper install elasticsearch
```

**自启动**：
查看系统是用的init 还是systemd  
ps -p 1

**如果是init**：  
sudo chkconfig --add elasticsearch
服务启停：
sudo -i service elasticsearch start
sudo -i service elasticsearch stop

**如果是systemd**：  
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
```
服务启停：
```
sudo systemctl start elasticsearch.service
sudo systemctl stop elasticsearch.service
```

服务启停是否异常，可以查看默认日志目录 ：/var/log/elasticsearch/ 下的日志:  

查看错误日志 tail journal:
```
sudo journalctl -f
```
To list journal entries for the elasticsearch service:
sudo journalctl --unit elasticsearch
To list journal entries for the elasticsearch service starting from a given time:
```
sudo journalctl --unit elasticsearch --since  "2016-10-30 18:17:16"
```

### 配置elasticsearch：
elasticsearch 默认监听9200，修改配置文件
/etc/elasticsearch

/elasticsearch.yml 配置日志，集群名称，节点名称， 日志目录和默认数据存储目录：

```
chown -R elasticsearch:elasticsearch /var/log/elasticsearch/
vim /etc/elasticsearch/elasticsearch.yml
```

找到配置文件中的cluster.name，打开该配置并设置集群名称
```
cluster.name: demon
```

找到配置文件中的node.name，打开该配置并设置节点名称
```
node.name: elk-1修改data存放的路径
path.data: /data/es-data修改logs日志的路径
path.logs: /var/log/elasticsearch/
```
配置内存使用用交换分区
```
bootstrap.memory_lock: true
```
监听的网络地址
```
network.host: 0.0.0.0开启监听的端口
http.port: 9200增加新的参数，这样head插件可以访问es (5.x版本，如果没有可以自己手动加)
http.cors.enabled: true
http.cors.allow-origin: "*"启动elasticsearch服务
```


### 运行elasticsearch:
对于yum安装的elasticsearch， 通过服务启动，启动脚本会自动修改limit，虚拟机存储大小等配置，可以不用设置系统的limit权限，如果是下载包安装的，没有启动脚本，手工启动时，需要以普通用户sudo启动，
- 普通用户启动  
如果不是以yum安装的,需要创建用户和用户组:  
groupadd elasticsearch  
useradd elasticsearch -g elasticsearch -p elasticsearch  

修改elasticsearch 日志和存储目录的权限.  
修改elasticsearch.yml日志
chown elasticsearch:elasticsearch /data/es-data/  


修改单个JVM下支撑100w线程数vm.max_map_count  
cat /proc/sys/vm/max_map_count  
永久修改：将vm.max_map_count=2048000配置到/etc/sysctl.conf中,然后执行sysctl -p生效，重启os后也会持久。  
也可以：
```sysctl -w vm.max_map_count=2048000```
注意：
- 在其他资源可用的前提下，单个JVM能开启的最大线程数是/proc/sys/vm/max_map_count的设置数的一半。  
- 异常描述为不能以root权限运行Elasticsearch.  
- 解决办法是运行时加上参数： `bin/elasticsearch -Des.insecure.allow.root=true`

- 修改系统limit:  
为用户名为elasticsearch的用户提高系统句柄数和进程数值:
```
vim /etc/security/limits.conf
在末尾追加以下内容（elk为启动用户，当然也可以指定为*）
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
elasticsearch soft nproc 2048
elasticsearch hard nproc 2048
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
```

另外还需注意一个问题（在日志发现如下内容，这样也会导致启动失败，这一问题困扰了很久）  
[2017-06-14T19:19:01,641][INFO ][o.e.b.BootstrapChecks    ] [elk-1] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks  
[2017-06-14T19:19:01,658][ERROR][o.e.b.Bootstrap          ] [elk-1] node validation exception  
[1] bootstrap checks failed  
[1]: system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk  
解决：修改配置文件，在配置文件添加一项参数（目前还没明白此参数的作用）  
vim /etc/elasticsearch/elasticsearch.yml  
bootstrap.system_call_filter: false  


也可以修改:  
vim /etc/sysctl.conf  
修改后生效  
sysctl -p  

- 启动服务
/etc/init.d/elasticsearch start  
./elasticsearch console  ------前台运行  
./elasticsearch start    ------后台运行  
./elasticsearch install  -------添加到系统自动启动  
./elasticsearch remove   -----取消随系统自动启动  
./elasticsearch stop     ------停止  
./elasticsearch restart  ------重新启动  


- 开机自启动  
chkconfig elasticsearch on

### 安装kibana：
- yum安装 三种安装命令:
```
sudo yum install kibana
sudo dnf install kibana
sudo zypper install kibana
```
init环境 用chkconfig配置服务自启动:
```
sudo chkconfig --add kibana
```
- 服务启动：
```
sudo -i service kibana start
sudo -i service kibana stop
```

- 用systemd编辑 配置Kibana 服务开机自启动:
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable kibana.service
```
- Kibana 服务启动:
```
sudo systemctl start kibana.service
sudo systemctl stop kibana.service
```



### 安装filebeat

- 安装命令(三种):
```
sudo yum install filebeat
sudo dnf install filebeat
sudo zypper install filebeat
```

1. Init环境下:  
Init环境下用chkconfig:
- 配置服务开机自启动:

```
sudo chkconfig --add filebeat
```
- 服务启动：
```
sudo -i service filebeat start
sudo -i service filebeat stop
```

2. systemd环境下:
- 配置服务开机自启动:
```
sudo /bin/systemctl daemon-
reload
sudo /bin/systemctl enable filebeat
```
- 服务启动:
```
sudo systemctl start filebeat
sudo systemctl stop filebeat
```






