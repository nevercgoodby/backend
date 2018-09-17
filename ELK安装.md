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
配置集群:  
集群之间通讯默认端口是9300. 集群名相同，则属于一个集群，另外为了防止脑裂，集群的最小数量，最好配置为集群节点总数/2 + 1
```
vim config/elasticsearch.yml
cluster.name: my-es
node.name: node-1
network.host: 192.168.19.141
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts: ["192.168.19.141","192.168.19.142","192.168.19.143"]
discovery.zen.minimum_master_nodes: 2
#避免出现跨域问题
http.cors.enabled: true
http.cors.allow-origin: "*"
第二个、第三个节点的配置只需修改成对应的ip即可。
```
### 运行elasticsearch:
对于yum安装的elasticsearch， 通过服务启动，启动脚本会自动修改limit，虚拟机存储大小等配置，可以不用设置系统的limit权限，如果是下载包安装的，没有启动脚本，手工启动时，需要以普通用户sudo启动，
- 普通用户启动  
ES有执行脚本的能力，因安全因素，不能在root用户下运行，强行运行会报如下错误：
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
解决方案：
如果是用yum安装,会在 init.d/下自动创建启动脚本，启动时自动指定普通用户启动。
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

## 扩展应用:

elasticsearch,底层依据于lucene搜索引擎,提供一系列基于restful的上层接口,可以方便的搭建一个分布式的全文检索系统,这里我们安装一个中文分词plugin,搭建一个基于中文的全文检索系统, 前提已经搭好一个node或一个cluster.

- elasticsearch 全文检索
    - 安装插件  
    中文分词插件是Ik, 关于IK分词器的介绍不再多少，一言以蔽之，IK分词是目前使用非常广泛分词效果比较好的中文分词器。做ES开发的，中文分词十有八九使用的都是IK分词器。
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip  
以普通用户运行失败,以root用户安装可以:   
/usr/share/elasticsearch/bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip  
安装完插件后,需要重新启动elasticsearch.
    - 创建索引  
curl -XPUT http://192.168.1.234:9200/index
    - 创建mapping  
curl -H 'Content-Type:application/json' http://192.168.1.234:9200/index/fulltext/_mapping -d'
{
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_max_word"
            }
        }
}'

    - 分词  
ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合；  
ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”。  
```
curl -H 'Content-Type:application/json' 'http://192.168.1.234:9200/index/_analyze?pretty=true' -d '
{
"text":"中华人民共和国",
"analyzer" : "ik_max_word"
}'

curl -XPOST http://localhost:9200/index/fulltext/1 -H 'Content-Type:application/json' -d'
{"content":"美国留给伊拉克的是个烂摊子吗"}
'
curl -XPOST http://localhost:9200/index/fulltext/2 -H 'Content-Type:application/json' -d'
{"content":"公安部：各地校车将享最高路权"}
'
curl -XPOST http://localhost:9200/index/fulltext/3 -H 'Content-Type:application/json' -d'
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
'
curl -XPOST http://localhost:9200/index/fulltext/4 -H 'Content-Type:application/json' -d'
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
'
```

测试分词效果:

`curl -XPOST "http://localhost:9200/index/_analyze?analyzer=ik_max_word&text=中华人民共和国"`
分词结果:
```
   {
    "tokens": [{
        "token": "中华人民共和国",
        "start_offset": 0,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 0
    }, {
        "token": "中华人民",
        "start_offset": 0,
        "end_offset": 4,
        "type": "CN_WORD",
        "position": 1
    }, {
        "token": "中华",
        "start_offset": 0,
        "end_offset": 2,
        "type": "CN_WORD",
        "position": 2
    }, {
        "token": "华人",
        "start_offset": 1,
        "end_offset": 3,
        "type": "CN_WORD",
        "position": 3
    }, {
        "token": "人民共和国",
        "start_offset": 2,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 4
    }, {
        "token": "人民",
        "start_offset": 2,
        "end_offset": 4,
        "type": "CN_WORD",
        "position": 5
    }, {
        "token": "共和国",
        "start_offset": 4,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 6
    }, {
        "token": "共和",
        "start_offset": 4,
        "end_offset": 6,
        "type": "CN_WORD",
        "position": 7
    }, {
        "token": "国",
        "start_offset": 6,
        "end_offset": 7,
        "type": "CN_CHAR",
        "position": 8
    }, {
        "token": "国歌",
        "start_offset": 7,
        "end_offset": 9,
        "type": "CN_WORD",
        "position": 9
    }]
}
```

使用ik_smart分词:

`curl -XPOST "http://localhost:9200/index/_analyze?analyzer=ik_smart&text=中华人民共和国"`
分词结果:
```
{
    "tokens": [{
        "token": "中华人民共和国",
        "start_offset": 0,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 0
    }, {
        "token": "国歌",
        "start_offset": 7,
        "end_offset": 9,
        "type": "CN_WORD",
        "position": 1
    }]
}
```




    - 拼音分词
    pinyin分词器的下载地址: 
https://github.com/medcl/elasticsearch-analysis-pinyin

测试拼音分词:

curl -XPOST "http://localhost:9200/index/_analyze?analyzer=pinyin&text=张学友"
分词结果:

{
    "tokens": [{
        "token": "zhang",
        "start_offset": 0,
        "end_offset": 1,
        "type": "word",
        "position": 0
    }, {
        "token": "xue",
        "start_offset": 1,
        "end_offset": 2,
        "type": "word",
        "position": 1
    }, {
        "token": "you",
        "start_offset": 2,
        "end_offset": 3,
        "type": "word",
        "position": 2
    }, {
        "token": "zxy",
        "start_offset": 0,
        "end_offset": 3,
        "type": "word",
        "position": 3
    }]
}

安装过程和IK一样，下载、打包、加入ES。这里不在重复上述步骤，给出最后配置截图 


五、IK+pinyin分词配置
5.1创建索引与分析器设置
创建一个索引，并设置index分析器相关属性:

curl -XPUT "http://localhost:9200/medcl/" -d'
{
    "index": {
        "analysis": {
            "analyzer": {
                "ik_pinyin_analyzer": {
                    "type": "custom",
                    "tokenizer": "ik_smart",
                    "filter": ["my_pinyin", "word_delimiter"]
                }
            },
            "filter": {
                "my_pinyin": {
                    "type": "pinyin",
                    "first_letter": "prefix",
                    "padding_char": " "
                }
            }
        }
    }
}'
创建一个type并设置mapping:

curl -XPOST http://localhost:9200/medcl/folks/_mapping -d'
{
    "folks": {
        "properties": {
            "name": {
                "type": "keyword",
                "fields": {
                    "pinyin": {
                        "type": "text",
                        "store": "no",
                        "term_vector": "with_positions_offsets",
                        "analyzer": "ik_pinyin_analyzer",
                        "boost": 10
                    }
                }
            }
        }
    }
}'
5.2索引测试文档
索引2份测试文档。 
文档1:

curl -XPOST http://localhost:9200/medcl/folks/andy -d'{"name":"刘德华"}'
文档2:

curl -XPOST http://localhost:9200/medcl/folks/tina -d'{"name":"中华人民共和国国歌"}'
5.3测试(1)拼音分词
下面四条命命令都可以匹配”刘德华”

curl -XPOST "http://localhost:9200/medcl/folks/_search?q=name.pinyin:liu"

curl -XPOST "http://localhost:9200/medcl/folks/_search?q=name.pinyin:de"

curl -XPOST "http://localhost:9200/medcl/folks/_search?q=name.pinyin:hua"

curl -XPOST "http://localhost:9200/medcl/folks/_search?q=name.pinyin:ldh"
5.4测试(2)IK分词测试
curl -XPOST "http://localhost:9200/medcl/_search?pretty" -d'
{
  "query": {
    "match": {
      "name.pinyin": "国歌"
    }
  },
  "highlight": {
    "fields": {
      "name.pinyin": {}
    }
  }
}'
返回结果:

{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 16.698704,
    "hits" : [
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "tina",
        "_score" : 16.698704,
        "_source" : {
          "name" : "中华人民共和国国歌"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>中华人民共和国</em><em>国歌</em>"
          ]
        }
      }
    ]
  }
}
说明IK分词器起到了效果。

5.3测试(4)pinyin+ik分词测试：
curl -XPOST "http://localhost:9200/medcl/_search?pretty" -d'
{
  "query": {
    "match": {
      "name.pinyin": "zhonghua"
    }
  },
  "highlight": {
    "fields": {
      "name.pinyin": {}
    }
  }
}'
返回结果:

{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 5.9814634,
    "hits" : [
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "tina",
        "_score" : 5.9814634,
        "_source" : {
          "name" : "中华人民共和国国歌"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>中华人民共和国</em>国歌"
          ]
        }
      },
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "andy",
        "_score" : 2.2534127,
        "_source" : {
          "name" : "刘德华"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>刘德华</em>"
          ]
        }
      }
    ]
  }
}
 

 

DELETE  /accounts
DELETE /pinyin
DELETE /pinyin2
DELETE /pinyin3
DELETE /pinyin4
DELETE /index

DELETE /accounts

GET /accounts

PUT /accounts
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}

POST /accounts/person/
{
  "user": "赵六",
  "title": "工程师",
  "desc": "软件开发"
}

GET accounts/person/_search


GET /accounts/person/_search
{
  "query" : { "match" : { "title" : "张" }}
}

#中文分词测试
DELETE /index

PUT /index

POST /index/fulltext/_mapping
{
  "properties": {
    "content":{
      "type" :"text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_max_word"
    }
  }
}

POST /index/_analyze?pretty=true
{
  "text": "美国留给伊拉克的是个烂摊子吗",
  "analyzer": "ik_max_word"
}

POST /index/_analyze?pretty=true
{
  "text": "美国留给伊拉克的是个烂摊子吗",
  "analyzer": "ik_smart"
}

# 拼音分词
DELETE /pinyin
PUT /pinyin/


PUT /pinyin/
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "pinyin_analyzer" : {
                    "tokenizer" : "my_pinyin"
                    }
            },
            "tokenizer" : {
                "my_pinyin" : {
                    "type" : "pinyin",
                    "keep_separate_first_letter" : true,
                    "keep_full_pinyin" : true,
                    "keep_original" : true,
                    "limit_first_letter_length" : 16,
                    "lowercase" : true,
                    "remove_duplicated_term" : true
                }
            }
        }
    }
}


GET /pinyin/_analyze
{
  "text": ["刘德华"],
  "analyzer": "pinyin_analyzer"
}

POST /pinyin/folks/andy
{
  "name": "刘德华"
}

POST /pinyin/folks/_mapping
{
    "folks": {
        "properties": {
            "name": {
                "type": "keyword",
                "fields": {
                    "pinyin": {
                        "type": "text",
                        "store": false,
                        "term_vector": "with_offsets",
                        "analyzer": "pinyin_analyzer",
                        "boost": 10
                    }
                }
            }
        }
    }
}


POST /pinyin/folks/andy
{
  "name": "刘德华"
}

GET /pinyin/folks/andy

GET /pinyin/folks/_search?q=name:%E5%88%98%E5%BE%B7%E5%8D%8E

GET /pinyin/folks/_search?q=name.pinyin:%e5%88%98%e5%be%b7

GET /pinyin/folks/_search?q=name.pinyin:liu
GET /pinyin/folks/_search?q=name.pinyin:de
GET /pinyin/folks/_search?q=name.pinyin:hua

GET /pinyin/folks/_search?q=name.pinyin:de+hua


PUT /pinyin2/ 
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "user_name_analyzer" : {
                    "tokenizer" : "whitespace",
                    "filter" : "pinyin_first_letter_and_full_pinyin_filter"
                }
            },
            "filter" : {
                "pinyin_first_letter_and_full_pinyin_filter" : {
                    "type" : "pinyin",
                    "keep_first_letter" : true,
                    "keep_full_pinyin" : false,
                    "keep_none_chinese" : true,
                    "keep_original" : false,
                    "limit_first_letter_length" : 16,
                    "lowercase" : true,
                    "trim_whitespace" : true,
                    "keep_none_chinese_in_first_letter" : true
                }
            }
        }
    }
}
POST /pinyin2/folks/andy
{
  "name": "刘德华"
}
GET /pinyin2/_analyze
{
  "text": ["刘德华 张学友 郭富城 黎明 四大天王"],
  "analyzer": "user_name_analyzer"
}


PUT /pinyin3/
{
  "index": {
    "analysis": {
      "analyzer": {
        "pinyin_analyzer": {
          "tokenizer": "my_pinyin"
        }
      },
      "tokenizer": {
        "my_pinyin": {
          "type": "pinyin",
          "keep_first_letter": false,
          "keep_separate_first_letter": false,
          "keep_full_pinyin": true,
          "keep_original": false,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      }
    }
  }
}
POST /pinyin3/folks/andy
{
  "name": "刘德华"
}
GET /pinyin3/folks/_search

GET /pinyin3/folks/_search
{
  "query": {
    "match_phrase": {
      "name.pinyin": "刘德华"
    }
  }
}

PUT /pinyin4/
{
  "index": {
    "analysis": {
      "analyzer": {
        "pinyin_analyzer": {
          "tokenizer": "my_pinyin"
        }
      },
      "tokenizer": {
        "my_pinyin": {
          "type": "pinyin",
          "keep_first_letter": false,
          "keep_separate_first_letter": true,
          "keep_full_pinyin": false,
          "keep_original": false,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      }
    }
  }
}

POST /pinyin4/folks/andy
{
  "name": "刘德华"
}

GET /pinyin4/folks/_search
{
  "query": {
    "match_phrase": {
      "name.pinyin": "刘德h"
    }
  }
}

GET /pinyin4/folks/_search
{
  "query": {
    "match_phrase": {
      "name.pinyin": "刘dh"
    }
  }
}

GET /pinyin4/folks/_search
{
  "query": {
    "match_phrase": {
      "name.pinyin": "dh"
    }
  }
}