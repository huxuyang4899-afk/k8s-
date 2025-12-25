#### ES 节点分类和规划
**1.Master 节点**

ES集群中只有一个 Master 节点，用于控制和管理整个集群的操作

Master 节点负责增删索引,增删节点,分片shard的重新分配

Master 主要维护**维护集群状态**（Cluster State），包括节点名称,节点连接地址,索引名称和配置信息等

当Cluster State有新数据产生后， Master 会将数据同步给其他 Node 节点

Master节点通过超过一半的节点投票选举产生的

可以设置node.master: true 指定为是否参与Master节点选举, 默认true

**2.Data 节点**

存储数据的节点即为 data 节点

当创建索引后，索引的数据会存储至某个数据节点

Data 节点消耗内存和磁盘IO的性能比较大

配置node.data: true, 就是Data节点，默认为 true,即默认所有节点都是 Data 节点类型

**3.Ingest 节点**

Ingest 节点是 Elasticsearch 5.0 新增的节点类型和功能。

如果集群中有大量数据预处理需求（如日志解析、字段提取等），可以引入专门的 Ingest 节点。

功能类似logstash

将 Ingest 节点与 Data 节点分离，避免数据预处理影响数据存储和查询性能。

负责数据预处理（如管道处理、数据转换等）

Ingest 节点的基础原理是：节点接收到数据之后，根据请求参数中指定的管道流 id，找到对应的已

注册管道流，对数据进行处理，然后将处理过后的数据，按照 Elasticsearch 标准的 indexing 流程

继续运行。

Ingest 节点开启方式为：在 elasticsearch.yml 中定义：node.ingest: true

**4.Coordinating 节点(协调)**

当集群规模较大时，建议引入专门的 Coordinating 节点。

Coordinating 节点不存储数据，仅负责接收客户端请求并分发到其他节点。

这可以减轻 Data 节点的负载，提高查询性能。

处理请求的节点即为 coordinating 节点，该节点类型为所有节点的默认角色，不能取消

coordinating 节点主要将请求路由到正确的节点处理。比如创建索引的请求会由 coordinating 路

由到 master 节点处理

当配置 node.master:false、node.data:false、node.ingest: false 则只充当 Coordinating 节点

Coordinating 节点在 Cerebro 等插件中数据页面不会显示出来

**5.Machine Learning 节点**

负责机器学习任务（如异常检测等）。

如果使用 Elasticsearch 的机器学习功能，可以配置专门的 Machine Learning 节点。

这些节点需要较高的 CPU 和内存资源。

Machine Learning节点开启方式为：在 elasticsearch.yml 中定义

node.ml: true (需要 enable x-pack)

**6.Master-eligible 初始化时有资格选举Master的节点**

集群初始化时有权利参于选举Master角色的节点

只在集群第一次初始化时进行设置有效，后续配置无效

由 cluster.initial_master_nodes 配置节点地址

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764940314617-082ab308-5195-435a-bc97-11c561b855f9.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764940308643-0594ec90-6d2f-483b-b2c2-b78283d14df5.png)

#### ES 文档路由
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764986514782-080de818-5e85-4084-98ff-5f7bcba0df01.png)

##### ES 文档创建删除流程
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764989045924-c656b2b9-fa48-4537-a956-ca2239281822.png)

客户端向集群中某个节点 Node1 发送新建索引文档或者删除索引文档请求

Node1节点使用文档的 _id 通过上面的算法确定文档属于分片 0

因为分片 0 的主分片目前被分配在 Node3 上,请求会被转发到 Node3

Node3 在主分片上面执行创建或删除请求

Node3 执行如果成功，它将请求并行转发到 Node1 和 Node2 的副本分片上

Node3 将向协调节点Node1 报告成功

协调节点Node1 客户端报告成功。

##### ES 文档读取流程
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764989115487-5cb09d17-3d9c-4acf-988f-b5e2758f7253.png)

客户端向集群中某个节点 Node1 发送读取请求

节点使用文档的 _id 来确定文档属于分片 0 。分片 0 的主副本分片存在于所有的三个节点上

在处理读取请求时，**协调节点**在每次请求的时候都会通过轮询所有的主副本分片来达到负载均衡，

此次它将请求转发到 Node2

Node2 将文档返回给 Node1 ，然后将文档返回给客户端

#### Elasticsearch 集群扩容和缩容
##### 集群扩容
新加入两个节点node4和node5，变为Data节点

在两个新节点安装 ES，并配置文件如下

```powershell
vim /etc/elasticsearch/elasticsearch.yml
cluster.name: ELK-Cluster # 和原集群名称相同
# 当前节点在集群内的节点名称，同一集群中每个节点要确保此名称唯一
node.name: es-node4 # 第二个新节点为es-node5
# 集群监听端口对应的IP，默认是127.0.0.1:9300
network.host: 0.0.0.0
# 指定任意集群节点即可
discovery.seed_hosts: ["10.0.0.101","10.0.0.102","10.0.0.103"]
# 集群初始化时指定希望哪些节点可以被选举为 master,只在初始化时使用,新加节点到已有集群时此项可不配置
# cluster.initial_master_nodes: ["10.0.0.101","10.0.0.102","10.0.0.103"]
# cluster.initial_master_nodes: ["ubuntu2204.wang.org"]
# 如果不参与主节点选举设为false,默认值为true
node.master: false
# 存储数据,默认值为true,此值为false则不存储数据而成为一个路由节点
# 如果将原有的true改为false,需要先执行/usr/share/elasticsearch/bin/elasticsearch-node repurpose 清理数据
node.data: true
systemctl restart elasticsearch
```

##### 缩容
从集群中删除两个节点node4和node5，在两个节点按一定的顺序逐个停止服务，即可自动退出集群

注意：停止服务前，要观察索引的情况，按一定顺序关机，即先关闭一台主机，等数据同步完成后，再

关闭第二台主机，防止数据丢失

systemctl stop elasticsearch

#### Elasticsearch 数据冷热分离
访问量多的是热数据--反之是冷数据

仅仅将不同的节点设置为不同的规格还不够，为了能明确区分出哪些节点是热节点，哪些节点是冷节

点，需要为对应节点打标签

Elasticsearch支持给节点打标签，具体方式是在elasticsearch.yml文件中增加

```powershell
node.attr.{attribute}: {value}
# 其中attribute为用户自定义的任意标签名，value为该节点对应的该标签的值
```

```powershell
# 例如对于冷热分离，可以使用如下设置
# 示例
vim elasticsearch.yml
node.attr.temperature: hot //热节点=====101 102 
node.attr.temperature: warm //冷节点=====103
# 验证
[root@ubuntu2404 ~]# curl 'http://10.0.0.101:9200/_cat/nodeattrs?v&h=node,attr,value&s=attr:desc'
```

指定数据的冷热属性，来设置和调整数据分布。冷热分离方案中数据冷热分布的基本单位是索引，即指

定某个索引为热索引，另一个索引为冷索引。通过索引的分布来实现控制数据分布的目的。

Elasticsearch提供了index shard filtering功能(2.x开始)，该功能在索引配置中提供了如下几个配置

用户可以在创建索引，或后续的任意时刻设置这些配置来控制索引在不同标签节点上的分配动作。

```powershell
index.routing.allocation.include.{attribute}  # 表示索引可以分配在包含多个逗号分隔的值中其中一个的节点上。
index.routing.allocation.require.{attribute}  # 表示索引要分配在包含索引指定多个逗号分隔的值的节点上,并且多个值都要匹配
index.routing.allocation.exclude.{attribute}  # 表示索引只能分配在不包含指定多个逗号分隔的值值的节点上。
```

```powershell
# 对于热数据，索引设置如下
PUT hot_warm_test_index/_settings
{
"index.routing.allocation.require.temperature": "hot"
}
# 示例:将index1索引设亩hot的属性
curl -XPUT http://10.0.0.101:9200/index1/_settings -H 'Content-Type:
application/json' -d '
{
"index.routing.allocation.require.temperature": "hot"
}'
# 查看分片分配,发现分片均分配在热节点上
GET _cat/shards/hot_warm_test_index?v&h=index,shard,prirep,node&s=node
# 查看index1的数据只存放在hot节点node-1和node-2节点上
[root@ubuntu2404 ~]# curl 'http://10.0.0.101:9200/_cat/shards/index1?v&h=index,shard,prirep,node&s=node'
index shard prirep node
index1 0 r node-1
index1 1 p node-1
index1 0 p node-2
index1 1 r node-2
# 对于冷数据，索引设置
PUT hot_warm_test_index/_settings
{
"index.routing.allocation.require.temperature": "warm"
}
# 查看分片分配，发现分片均分配到冷节点上
GET _cat/shards/hot_warm_test_index?v&h=index,shard,prirep,node&s=node
```

#### Kibana 图形显示
Kibana 官方下载链接:

```powershell
https://www.elastic.co/cn/downloads/kibana
https://www.elastic.co/cn/downloads/past-releases#kibana
镜像网站下载链接:
https://mirrors.tuna.tsinghua.edu.cn/elasticstack/
官方demo
https://demo.elastic.co/
```

可以通过包或者二进制的方式进行安装,可以安装在独立服务器,或者也可以和elasticsearch的主机安装在

一起 注意: Kibana的版本要和 Elasticsearch 相同的版本，否则可能会出错

```powershell
# dpkg -i kibana-9.2.2-amd64.deb
# dpkg -l kibana
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version      Architecture Description
+++-==============-============-============-=============================================
ii  kibana         9.2.2        amd64        Explore and visualize your Elasticsearch data

[root@es-node1 ~]# vim /etc/kibana/kibana.yml
[root@es-node1 ~]# grep "^[a-Z]" /etc/kibana/kibana.yml
server.port: 5601 #监听端口,此为默认值,可不做修改
server.host: "0.0.0.0" #修改此行的监听地址,默认为localhost，即：127.0.0.1:5601
# 修改此行,指向ES任意服务器地址或多个节点地址实现容错,默认为localhost
elasticsearch.hosts: ["http://10.0.0.101:9200","http://10.0.0.102:9200","http://10.0.0.103:9200"]
i18n.locale: "zh-CN" # 修改此行,使用"zh-CN"显示中文界面,默认英文
# 8.X版本新添加配置,默认被注释,会显示下面提示
server.publicBaseUrl: "http://kibana.wang.org"
```



![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1764993100032-58699795-bdfc-4f99-8817-cef3e48ae05a.png)

#### Beats 收集数据
虽然利用 logstash 就可以收集日志，功能强大，但由于 Logstash 是基于Java实现，需要在采集日志的

主机上安装JAVA环境

logstash运行时最少也会需要额外的500M的以上的内存，会消耗比较多的内存和磁盘空间，

可以采有基于Go开发的 Beat 工具代替 Logstash 收集日志，部署更为方便，而且只占用10M左右的内

存空间及更小的磁盘空间。

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765004321049-2bb5679c-62bd-4b60-91b7-f63dbd95bff0.png)

##### 利用 Metricbeat 监控性能相关指标
```powershell
# 新版下载
[root@elk-web2 ~]#wget https://mirrors.tuna.tsinghua.edu.cn/elasticstack/8.x/apt/pool/main/m/metricbeat/metricbeat-8.6.1-amd64.deb
[root@elk-web2 ~]# vim /etc/metricbeat/metricbeat.yml
# setup.kibana:
# host: "10.0.0.101:5601" #指向kabana服务器地址和端口，非必须项，即使不设置Kibana也可
以通过ES获取Metrics信息
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
# Array of hosts to connect to.
hosts: ["10.0.0.101:9200","10.0.0.102:9200","10.0.0.103:9200"] #指向任意一个ELK集群节点即可
[root@elk-web2 ~]# grep -Ev "#|^$" /etc/metricbeat/metricbeat.yml
metricbeat.config.modules:
path: ${path.config}/modules.d/*.yml
reload.enabled: false
setup.template.settings:
index.number_of_shards: 1
index.codec: best_compression
setup.kibana:
host: "10.0.0.101:5601"
output.elasticsearch:
hosts: ["10.0.0.101:9200","10.0.0.102:9200","10.0.0.103:9200"]
processors:
- add_host_metadata: ~
- add_cloud_metadata: ~
- add_docker_metadata: ~
- add_kubernetes_metadata: ~
# systemctl enable --now metricbeat.service
```

##### 利用 Heartbeat 监控
```powershell
下载链接
https://www.elastic.co/cn/downloads/beats/heartbeat
https://mirrors.tuna.tsinghua.edu.cn/elasticstack/8.x/apt/pool/main/h/heartbeat-
elastic/
[root@elk-web2 ~]#vim /etc/heartbeat/heartbeat.yml
# Configure monitors inline
heartbeat.monitors:
- type: http
enabled: true #修改此行false为true
# List or urls to query
urls: ["http://10.0.0.105"] #修改此行，指向需要监控的服务器的地址和端口
# Configure task schedule
schedule: '@every 10s'
# Total test connection and data exchange timeout
timeout: 6s

# 添加下面内容，包括 tcp 和 icmp
- type: tcp
  id: myhost-tcp-echo
  name: My Host TCP Echo
  hosts: ["10.0.0.105:80"] # default TCP Echo Protocol
  schedule: '@every 5s'
- type: icmp #添加下面5行，用于监控ICMP
  id: ping-myhost
  name: My Host Ping
  hosts: ["10.0.0.105"]
  schedule: '*/5 * * * * * *'
  
setup.kibana:
# Kibana Host
# Scheme and port can be left out and will be set to the default (http and
5601)
# In case you specify and additional path, the scheme is required:
http://localhost:5601/path
# IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
host: "10.0.0.101:5601 #指向kibana服务器地址和端口,可选
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
# Array of hosts to connect to.
hosts: ["10.0.0.101:9200"] #修改此行，指向ELK集群服务器地址和端口

[root@elk-web2 ~]#systemctl enable --now heartbeat-elastic.service

105 主机安装nginx 以进行测试
```

kibana 视图查看数据

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765009513201-a6dd9090-bf15-4857-8c89-9e2949adea5f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765009539372-81d7cfc0-5515-4c62-9514-ef1147c49371.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765009659429-2d4db81c-f921-495b-96fa-20cdbf9c796c.png)

##### 利用 Filebeat 收集日志
Filebeat 是用于转发和集中日志数据的轻量级传送程序。作为服务器上的代理安装，Filebeat监视您指定

的日志文件或位置，收集日志事件，并将它们转发到Elasticsearch或Logstash进行索引。

Logstash 也可以直接收集日志,但需要安装JDK并且会占用至少 500M 以上的内存

生产一般使用filebeat代替logstash, 基于go开发,部署方便,重要的是只需要10M多内存,比较节约资源.

filebeat 支持从日志文件,Syslog,Redis,Docker,TCP,UDP,标准输入等读取数据,对数据做简单处理，再输

出至Elasticsearch,logstash,Redis,Kafka等

Filebeat的工作方式如下：

启动Filebeat时，它将启动一个或多个输入源，这些输入将在为日志数据指定的位置中查找。

对于Filebeat所找到的每个日志，**Filebeat都会启动收集器harvester进程**。

每个收集器harvester都读取一个日志以获取新内容，并将新日志数据发送到libbeat；libbeat会汇总事件并将汇总的数据发送到为Filebeat配置的输出。

**注意: Filebeat 支持多个输入,但不支持同时有多个输出，如果多输出，会报错如下**

```powershell
# 下载链接
https://www.elastic.co/cn/downloads/beats/filebeat
https://mirrors.tuna.tsinghua.edu.cn/elasticstack/8.x/apt/pool/main/f/filebeat/
#安装
[root@elk-web1 ~]#dpkg -i filebeat-9.2.2-amd64.deb
```

###### 案例: 从标准输入读取再输出至标准输出
```powershell
[root@elk-web1 ~]#vim /etc/filebeat/stdin.yml
filebeat.inputs:
- type: stdin
  enabled: true
  tags: ["stdin-tags","myapp"] #添加新字段名tags，可以用于判断不同类型的输入，实现不同的输出
  fields:
    status_code: "200" #添加新字段名fields.status_code，可以用于判断不同类型的输入，实现不同的输出
    author: "wangxiaochun"
output.console:
  pretty: true
  enable: true
# 语法检查，注意：stdin.yml的相对路径是相对于/etc/filebeat的路径，而不是当前路径
[root@ubuntu2204 ~]# filebeat test config -c stdin.yml
Config OK
[root@web02 ~]#filebeat -c stdin.yml
在屏幕输入信息就可以输出到屏幕--不支持输入json格式
```

###### 从标准输入读取再输出至 Json 格式的文件
```powershell
[root@elk-web1 ~]#vim /etc/filebeat/stdout_file.yml
filebeat.inputs:
- type: stdin
  enabled: true
  json.keys_under_root: true #默认False会将json数据存储至message，true则会将数据以独立字段存储,并且删除message字段，如果是文本还是放在message字段中
output.file:
  path: "/tmp"
  filename: "filebeat.log"
```

###### 从文件读取再输出至标准输出
[https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-filestream](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-filestream)

[https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html)

filebeat 的INPUT中支持log(旧版)和filestream(新版)两种方式读取文件

filebeat 会将每个文件的读取数据的相关信息记录在/var/lib/filebeat/registry/filebeat/log.json文

件中,可以实现日志采集的持续性,而不会重复采集

当日志文件大小发生变化时，filebeat会接着上一次记录的位置继续向下读取新的内容

当日志文件大小没有变化，但是内容发生变化，filebeat会将文件的全部内容重新读取一遍

```powershell
[root@ubuntu2204 ~]#cat /etc/filebeat/file.yml
filebeat.inputs:
- type: filestream
  id: my-filestream-id
  enabled: true
  #tags: ["myapp"]
  #fields:
  # app_id: myappid
paths:
  - /var/log/test.log #注意:文件不能太小,要求超过1024字节
parsers:
  - ndjson:
    target: "" #解析结果存放在指定字段下,如果为空,则保存在根下
    #message_key: message #对哪个字段做json解析,可选
    
output.console:
  pretty: true
  enable: true

# 输入json解析出来的结果
[root@ubuntu2204 ~]# echo '{"name": "wangxiaochun", "age": 18, "phone":"0123456789"}' >> /var/log/test.log
# 观察结果
[root@ubuntu2204 ~]# filebeat -c /etc/filebeat/file.yml
```

###### 利用 Filebeat 收集系统日志到 ELasticsearch
```powershell
filebeat.inputs:
- type: stdin
  json.keys_under_root: true
  json.add_error_key: true
  json.message_key: log
  tags: ["stdin-tags","myapp"] #添加新字段名tags，可以用于判断不同类型的输入，实现不同的输出
  fields:
    status_code: "200" #添加新字段名fields.status_code，可以用于判断不同类型的输入，实现不同的输出
    author: "wangxiaochun"
#output.console:
 # pretty: true
output.elasticsearch:
  hosts: ["http://10.0.0.101:9200"]

# 测试
filebeat -c stdin-es.yml
```

###### 自定义索引名称收集日志到 ELasticsearch
```powershell
filebeat.inputs:
  - type: filestream
    id: my-filestream-id
    enabled: true

    tags: ["nginx-access"]
    fields:
      log: nginx-access
    paths:
      - /var/log/nginx/access.log    # 注意：文件不能太小，要求超过1024字节
    parsers:
      - ndjson:
          target: ""    # 解析结果存放在指定字段下，如果为空，则保存在根下
          # message_key: message    # 对哪个字段做json解析，可选

# --- Elasticsearch output ---
output.elasticsearch:
  hosts: ["10.0.0.101:9200","10.0.0.102:9200","10.0.0.103:9200"] # 指定ELK集群任意节点的地址和端口，多个地址容错
  index: "nginxaccesslog-%{[agent.version]}-%{+yyyy.MM.dd}"    # 自定义索引名称，8.x的索引为.ds-wang，agent.version是filebeat添加的元数据字段
  # 注意：8.x版生成的索引名为.ds-wang-%{[agent.version]}-日期-日期-00000n
  # 注意：如果自定义索引名称，没有添加下面三行的配置会导致filebeat无法启动，提示错误日志如下：
  # filebeat: Exiting: setup.template.name and setup.template.pattern have to be set if index name is modified
  # systemd: filebeat.service: main process exited, code=exited, status=1/FAILURE

setup.ilm.enabled: false    # 关闭索引生命周期ILM功能，默认开启时索引名称只能为filebeat-x，自定义索引名必须修改为false
setup.template.name: "nginxaccesslog"    # 定义模板名称，要自定义索引名称，必须指定此项，否则无法启动
setup.template.pattern: "nginxaccesslog-*"    # 定义模板的匹配索引名称，要自定义索引名称，必须指定此项，否则无法启动
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1

```

kibana 添加数据视图

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765022691879-913b792f-235d-44aa-b745-5b3b63e84de0.png)

104  105 主机同时安装 nginx 并且 filebeat 配置文件一样 那么 kibana 的一个数据视图就有 2 个节点的日志数据

###### 利用 tags 收集 Nginx的 Json 格式访问日志和错误日志到 Elasticsearch 不同的索引
 nginx 配置访问日志使用 Json 格式

```powershell
# 修改nginx访问日志为Json格式
[root@elk-web1 ~]# vim /etc/nginx/nginx.conf
.....
log_format access_json '{"@timestamp":"$time_iso8601",'
        '"host":"$server_addr",'
        '"clientip":"$remote_addr",'
        '"size":$body_bytes_sent,'
        '"responsetime":$request_time,'
        '"upstreamtime":"$upstream_response_time",'
        '"upstreamhost":"$upstream_addr",'
        '"http_host":"$host",'
        '"uri":"$uri",'
        '"domain":"$host",'
        '"xff":"$http_x_forwarded_for",'
        '"referer":"$http_referer",'
        '"tcp_xff":"$proxy_protocol_addr",'
        '"http_user_agent":"$http_user_agent",'
        '"status":"$status"}';
access_log /var/log/nginx/access_json.log access_json ;
#access_log /var/log/nginx/access.log;
```

```powershell
filebeat.inputs:
  - type: filestream
    id: my-filestream-id
    enabled: true

    tags: ["nginx-access"]
    fields:
      log: nginx-access
      author: "huxuyang"
    paths:
     # - /var/log/nginx/access.log    # 注意：文件不能太小，要求超过1024字节
      - /var/log/nginx/access_json.log    # 注意：文件不能太小，要求超过1024字节
    parsers:
      - ndjson:
          target: ""    # 解析结果存放在指定字段下，如果为空，则保存在根下
          # message_key: message    # 对哪个字段做json解析，可选

# --- Elasticsearch output ---
output.elasticsearch:
  hosts: ["10.0.0.101:9200","10.0.0.102:9200","10.0.0.103:9200"] # 指定ELK集群任意节点的地址和端口，多个地址容错
  index: "nginxaccesslog-%{[agent.version]}-%{+yyyy.MM.dd}"    # 自定义索引名称，8.x的索引为.ds-wang，agent.version是filebeat添加的元数据字段
  # 注意：8.x版生成的索引名为.ds-wang-%{[agent.version]}-日期-日期-00000n
  # 注意：如果自定义索引名称，没有添加下面三行的配置会导致filebeat无法启动，提示错误日志如下：
  # filebeat: Exiting: setup.template.name and setup.template.pattern have to be set if index name is modified
  # systemd: filebeat.service: main process exited, code=exited, status=1/FAILURE

setup.ilm.enabled: false    # 关闭索引生命周期ILM功能，默认开启时索引名称只能为filebeat-x，自定义索引名必须修改为false
setup.template.name: "nginxaccesslog"    # 定义模板名称，要自定义索引名称，必须指定此项，否则无法启动
setup.template.pattern: "nginxaccesslog-*"    # 定义模板的匹配索引名称，要自定义索引名称，必须指定此项，否则无法启动
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1
```

###### 利用 tags 收集 Nginx的 Json 格式访问日志和错误日志到 Elasticsearch 不同的索引
```powershell
[root@web01 ~]# cat /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: filestream
    id: my-filestream-id-1
    enabled: true
    tags: ["nginx-access"] # 指定tag,用于分类
    paths:
      - /var/log/nginx/access_json.log # 注意:文件不能太小,要求超过1024字节
    parsers:
      - ndjson:
          target: "" # 解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选
  - type: filestream
    id: my-filestream-id-2
    enabled: true
    tags: ["nginx-error"]
    paths:
      - /var/log/nginx/error.log # 注意:文件不能太小,要求超过1024字节
    parsers:
      - ndjson:
          target: "" # 解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选

output.elasticsearch:
  hosts: ["10.0.0.101:9200", "10.0.0.102:9200", "10.0.0.103:9200"]
  indices:
    - index: "nginx-access-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "nginx-access" # 如果记志中有access的tag,就记录到nginx-access的索引中
    - index: "nginx-error-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "nginx-error" # 如果记志中有error的tag,就记录到nginx-error的索引中

setup.ilm.enabled: false #关闭索引生命周期ilm功能，默认开启时索引名称只能为filebeat-*
setup.template.name: "nginx" # 定义模板名称,要自定义索引名称,必须指定此项,否则无法启动
setup.template.pattern: "nginx-*" #定义模板的匹配索引名称,要自定义索引名称,必须指定此项,否则无法启动
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765028278573-6d7605bd-fa38-4c23-9155-843e6e104f8b.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765028298337-b67134b8-1dcc-4fe0-8083-85904821dbd5.png)

###### 利用 Filebeat 收集 Tomat 的 Json 格式的访问日志和错误日志到 Elasticsearch
安装 Tomcat 并配置使用 Json 格式的访问日志

```powershell
# 安装Tomcat,可以包安装或者二进制安装
[root@ubuntu2404 ~]# apt update && apt -y install tomcat10
# 更改日志格式
pattern="{&quot;clientip&quot;:&quot;%h&quot;,&quot;ClientUser&quot;:&quot;%l&quot;,&quot;authenticated&quot;:&quot;%u&quot;,&quot;AccessTime&quot;:&quot;%t&quot;,&quot;method&quot;:&quot;%r&quot;,&quot;status&quot;:&quot;%s&quot;,&quot;SendBytes&quot;:&quot;%b&quot;,&quot;QueryString&quot;:&quot;%q&quot;,&quot;partner&quot;:&quot;%{Referer}i&quot;,&quot;AgentVersion&quot;:&quot;%{User-Agent}i&quot;}"
# systemctl restart tomcat10.service
```

修改 filebeat 文件

```powershell
cat /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: filestream
    id: my-filestream-id-1
    enabled: true
    tags: ["tomcat-access"]
    paths:
      #- /usr/local/tomcat/logs/localhost_access_log.* #二进制安装
      #- /var/log/tomcat9/localhost_access_log.* #包安装
      - /var/log/tomcat10/localhost_access_log.* #包安装 #注意:文件不能太小,要求超过1024字节
    parsers:
      - ndjson:
          target: "" #解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选
  - type: filestream
    id: my-filestream-id-2
    enabled: true
    tags: ["tomcat-error"]
    paths:
      #- /usr/local/tomcat/logs/catalina.*.log #二进制安装
      #- /var/log/tomcat9/catalina.*.log #包安装
      - /var/log/tomcat10/catalina.*.log #包安装
    parsers:
      - ndjson:
          target: "" #解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选
output.elasticsearch:
  hosts: ["10.0.0.101:9200"] #指定ELK集群服务器地址和端口
  indices:
    - index: "tomcat-access-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "tomcat-access"
    - index: "tomcat-error-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "tomcat-error"
setup.ilm.enabled: false
setup.template.name: "tomcat"
setup.template.pattern: "tomcat-*"
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1
```

###### 利用 Filebeat 收集 Tomat 的多行错误日志到Elasticsearch
Tomcat 是 Java 应用,当只出现一个错误时,会显示很多行的错误日志

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765092483073-3762ebd3-3830-4aed-a676-9b6fd4bdec51.png)

Java 应用的一个错误导致生成的多行日志其实是同一个事件的日志的内容

而ES默认是根据每一行来区别不同的日志,就会导致一个错误对应多行错误信息会生成很多行的ES文档记

录可以将一个错误对应的多个行合并成一个ES的文档记录来解决此问题

[https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html](https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html)

[https://www.elastic.co/guide/en/beats/filebeat/7.0/multiline-examples.html](https://www.elastic.co/guide/en/beats/filebeat/7.0/multiline-examples.html)![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765092846895-9d1ab9fb-1d4b-49c6-a693-656a15417c47.png)

ex：以 b 开头的行 ^b  
negate 表示不匹配 XXX ====所以当它为 false；就是双重否定为肯定 就是匹配 b 开头的行

negate 表示不匹配 XXX ====所以当它为 true；就是不匹配b 开头的行

所以此时 filebeatd 的配置文件

```powershell
filebeat.inputs:
  - type: filestream
    id: my-filestream-id-1
    enabled: true
    tags: ["tomcat-access"]
    paths:
      #- /usr/local/tomcat/logs/localhost_access_log.* #二进制安装
      #- /var/log/tomcat9/localhost_access_log.* #包安装
      - /var/log/tomcat10/localhost_access_log.* #包安装 #注意:文件不能太小,要求超过1024字节
    parsers:
      - ndjson:
          target: "" #解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选
  - type: filestream
    id: my-filestream-id-2
    enabled: true
    tags: ["tomcat-error"]
    paths:
      #- /usr/local/tomcat/logs/catalina.*.log #二进制安装
      #- /var/log/tomcat9/catalina.*.log #包安装
      - /var/log/tomcat10/catalina.*.log #包安装
    parsers:
      - multiline:
          type: pattern
          pattern: '^[0-3][0-9]-'
          negate: true
          match: after
      - ndjson:
          target: "" #解析结果存放在指定字段下,如果为空,则保存在根下
          #message_key: message #对哪个字段做json解析,可选
output.elasticsearch:
  hosts: ["10.0.0.101:9200"] #指定ELK集群服务器地址和端口
  indices:
    - index: "tomcat-access-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "tomcat-access"
    - index: "tomcat-error-%{[agent.version]}-%{+yyyy.MM.dd}"
      when.contains:
        tags: "tomcat-error"
setup.ilm.enabled: false
setup.template.name: "tomcat"
setup.template.pattern: "tomcat-*"
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1

scp /etc/filebeat/filebeat.yml root@10.0.0.105:/etc/filebeat/filebeat.yml
systemctl restart filebeat.service  
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765094014577-87e4d7b1-845a-4e34-a6eb-8a6aacdf986f.png)

###### 利用 Filebeat 收集 Nginx 日志到 Redis
前面已经配置Nginx 配置访问日志使用 Json格式

```powershell
[root@ubuntu2404 ~]# apt -y install redis
[root@ubuntu2404 ~]# sed -i.bak '/^bind.*/c bind 0.0.0.0' /etc/redis/redis.conf
[root@ubuntu2404 ~]# echo "requirepass 123456" >> /etc/redis/redis.conf
[root@ubuntu2404 ~]# systemctl restart redis
```

修改 Filebeat 配置文件

```powershell
filebeat.inputs:
- type: filestream
  id: my-filestream-id-1
  enabled: true
  tags: ["tomcat-access"]  
  paths:
    #- /usr/local/tomcat/logs/localhost_access_log.*  #二进制安装
    #- /var/log/tomcat9/localhost_access_log.*         #包安装
    - /var/log/tomcat10/localhost_access_log.*         #包安装  #注意:文件不能太小,要求超过1024字节   
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选

- type: filestream
  id: my-filestream-id-2
  enabled: true
  tags: ["tomcat-error"]
  paths:
    #- /usr/local/tomcat/logs/catalina.*.log  #二进制安装
    #- /var/log/tomcat9/catalina.*.log       #包安装
    - /var/log/tomcat10/catalina.*.log       #包安装    
  parsers:
  - multiline:
      type: pattern
      pattern: '^[0-3][0-9]-'
      negate: true
      match: after
  - ndjson:
      target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
      #message_key: message   #对哪个字段做json解析,可选

output.redis:
  hosts: ["10.0.0.200"]
  password: "123456"
  key: "m65-tomcat"
  db: 0
  timeout: 5
```

```powershell
[root@ubuntu2404 ~]# redis-cli -a 123456
127.0.0.1:6379> dbsize
(integer) 1
127.0.0.1:6379> keys *
1) "m65-tomcat"
127.0.0.1:6379> type m65-tomcat  ====确认数据类型
list
127.0.0.1:6379> LRANGE m65-tomcat 0 -1  ====列表数据查看命令
1) "{\"@timestamp\":\"2025-12-07T08:06:28.283Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"SendBytes\":\"1905\",\"method\":\"GET / HTTP/1.1\",\"QueryString\":\"\",\"ClientUser\":\"-\",\"partner\":\"-\",\"input\":{\"type\":\"filestream\"},\"clientip\":\"10.0.0.101\",\"log\":{\"offset\":0,\"file\":{\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\",\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\"}},\"AccessTime\":\"[07/Dec/2025:15:09:28 +0800]\",\"ecs\":{\"version\":\"8.0.0\"},\"host\":{\"name\":\"web01\"},\"authenticated\":\"-\",\"agent\":{\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\",\"type\":\"filebeat\",\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\"},\"status\":\"200\",\"AgentVersion\":\"curl/8.5.0\",\"tags\":[\"tomcat-access\"]}"
2) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"status\":\"200\",\"ClientUser\":\"-\",\"clientip\":\"10.0.0.100\",\"host\":{\"name\":\"web01\"},\"agent\":{\"type\":\"filebeat\",\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\",\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\"},\"ecs\":{\"version\":\"8.0.0\"},\"partner\":\"-\",\"QueryString\":\"\",\"AgentVersion\":\"curl/8.5.0\",\"tags\":[\"tomcat-access\"],\"authenticated\":\"-\",\"SendBytes\":\"1905\",\"input\":{\"type\":\"filestream\"},\"log\":{\"offset\":226,\"file\":{\"inode\":\"6292893\",\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\",\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\"}},\"AccessTime\":\"[07/Dec/2025:15:18:52 +0800]\",\"method\":\"GET / HTTP/1.1\"}"
3) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"AgentVersion\":\"curl/8.5.0\",\"ClientUser\":\"-\",\"authenticated\":\"-\",\"method\":\"GET / HTTP/1.1\",\"status\":\"200\",\"input\":{\"type\":\"filestream\"},\"AccessTime\":\"[07/Dec/2025:15:18:57 +0800]\",\"clientip\":\"10.0.0.100\",\"agent\":{\"name\":\"web01\",\"type\":\"filebeat\",\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\",\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\"},\"log\":{\"offset\":452,\"file\":{\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\",\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\"}},\"SendBytes\":\"1905\",\"ecs\":{\"version\":\"8.0.0\"},\"tags\":[\"tomcat-access\"],\"host\":{\"name\":\"web01\"},\"QueryString\":\"\",\"partner\":\"-\"}"
4) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"clientip\":\"10.0.0.100\",\"QueryString\":\"\",\"tags\":[\"tomcat-access\"],\"input\":{\"type\":\"filestream\"},\"ecs\":{\"version\":\"8.0.0\"},\"log\":{\"offset\":678,\"file\":{\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\",\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\"}},\"method\":\"GET / HTTP/1.1\",\"SendBytes\":\"1905\",\"partner\":\"-\",\"agent\":{\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\",\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\",\"type\":\"filebeat\"},\"authenticated\":\"-\",\"host\":{\"name\":\"web01\"},\"AgentVersion\":\"curl/8.5.0\",\"status\":\"200\",\"AccessTime\":\"[07/Dec/2025:15:18:58 +0800]\",\"ClientUser\":\"-\"}"
5) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"log\":{\"offset\":904,\"file\":{\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\",\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\"}},\"authenticated\":\"-\",\"ClientUser\":\"-\",\"status\":\"200\",\"QueryString\":\"\",\"AgentVersion\":\"curl/8.5.0\",\"clientip\":\"10.0.0.101\",\"agent\":{\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\",\"type\":\"filebeat\",\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\"},\"partner\":\"-\",\"AccessTime\":\"[07/Dec/2025:16:06:20 +0800]\",\"input\":{\"type\":\"filestream\"},\"ecs\":{\"version\":\"8.0.0\"},\"method\":\"GET / HTTP/1.1\",\"tags\":[\"tomcat-access\"],\"host\":{\"name\":\"web01\"},\"SendBytes\":\"1905\"}"
6) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"authenticated\":\"-\",\"QueryString\":\"\",\"ecs\":{\"version\":\"8.0.0\"},\"host\":{\"name\":\"web01\"},\"agent\":{\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\",\"type\":\"filebeat\",\"version\":\"9.2.2\",\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\"},\"AgentVersion\":\"curl/8.5.0\",\"log\":{\"offset\":1130,\"file\":{\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\",\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\"}},\"status\":\"200\",\"SendBytes\":\"1905\",\"tags\":[\"tomcat-access\"],\"partner\":\"-\",\"clientip\":\"10.0.0.101\",\"input\":{\"type\":\"filestream\"},\"method\":\"GET / HTTP/1.1\",\"ClientUser\":\"-\",\"AccessTime\":\"[07/Dec/2025:16:06:20 +0800]\"}"
7) "{\"@timestamp\":\"2025-12-07T08:06:28.284Z\",\"@metadata\":{\"beat\":\"filebeat\",\"type\":\"_doc\",\"version\":\"9.2.2\"},\"ClientUser\":\"-\",\"AccessTime\":\"[07/Dec/2025:16:06:21 +0800]\",\"status\":\"200\",\"input\":{\"type\":\"filestream\"},\"partner\":\"-\",\"AgentVersion\":\"curl/8.5.0\",\"SendBytes\":\"1905\",\"authenticated\":\"-\",\"tags\":[\"tomcat-access\"],\"method\":\"GET / HTTP/1.1\",\"ecs\":{\"version\":\"8.0.0\"},\"host\":{\"name\":\"web01\"},\"QueryString\":\"\",\"log\":{\"offset\":1356,\"file\":{\"fingerprint\":\"fe24f81d61926666d143e3640d6902c4410ec36f47bb743fa308956970703219\",\"path\":\"/var/log/tomcat10/localhost_access_log.2025-12-07.txt\",\"device_id\":\"64512\",\"inode\":\"6292893\"}},\"clientip\":\"10.0.0.101\",\"agent\":{\"ephemeral_id\":\"de2c4753-975c-4aae-b68f-bed7860444f2\",\"id\":\"4aa85dc6-ecc6-4558-8c36-3a668ef19318\",\"name\":\"web01\",\"type\":\"filebeat\",\"version\":\"9.2.2\"}}"

```

###### 从标准输入读取再输出至 Kafka
```powershell
[root@ubuntu2404 ~]#ls
alyun.sh  kafka_2.13-4.1.1.tgz
[root@ubuntu2404 ~]#tar xf kafka_2.13-4.1.0.tgz -C /usr/local/
tar: kafka_2.13-4.1.0.tgz: Cannot open: No such file or directory
tar: Error is not recoverable: exiting now
[root@ubuntu2404 ~]#tar xf kafka_2.13-4.1.1.tgz -C /usr/local/
[root@ubuntu2404 ~]#cd /usr/local/
[root@ubuntu2404 local]#ls
bin  etc  games  include  kafka_2.13-4.1.1  lib  man  sbin  share  src
[root@ubuntu2404 local]#ln -s kafka_2.13-4.1.1/ kafka
[root@ubuntu2404 local]#KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
-bash: bin/kafka-storage.sh: No such file or directory
[root@ubuntu2404 local]#cd kafka
[root@ubuntu2404 kafka]#KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
[root@ubuntu2404 kafka]#echo $KAFKA_CLUSTER_ID
l1qCw9UvT1u6xnE0mhxpfA
[root@ubuntu2404 kafka]#bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
Formatting dynamic metadata voter directory /tmp/kraft-combined-logs with metadata.version 4.1-IV1.
[root@ubuntu2404 kafka]#bin/kafka-server-start.sh config/server.properties
```

```powershell
filebeat.inputs:
- type: filestream
  id: my-filestream-id-1
  enabled: true
  tags: ["nginx-access"]     #指定tag,用于分类  
  paths:
    - /var/log/nginx/access_json.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选
- type: filestream
  id: my-filestream-id-2
  enabled: true
  tags: ["nginx-error"]  
  paths:
    - /var/log/nginx/error.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选


output.kafka:
  hosts: ["10.0.0.201:9092", "10.0.0.202:9092", "10.0.0.203:9092"]
  topic: filebeat-log       #指定kafka的topic
  partition.round_robin:
    reachable_only: true    #true表示只发布到可用的分区，不可用分区就不发布，false 时表示所有分区，如果一个节点down，会block
  required_acks: 1          #如果为0，不启用确认机制，错误消息可能会丢失，1等待写入主分区（默认），-1等待写入副本分区
  compression: gzip  
  max_message_bytes: 1000000 #每条消息最大长度，以字节为单位，如果超过将丢弃
```

有消息生成

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765097644419-c65e1986-0e8b-4aaf-beb3-b44c2a024205.png)

#### Logstash 过滤
#9.X要求安装JDK17或21,9.X版logstash包已经内置了JDK无需安装

```powershell
[root@logstash ~]#dpkg -i logstash-8.6.1-amd64.deb
官方文档
https://www.elastic.co/guide/en/logstash/current/first-event.html
#各种插件
https://www.elastic.co/guide/en/logstash/current/input-plugins.html
https://www.elastic.co/guide/en/logstash/current/filter-plugins.html
https://www.elastic.co/guide/en/logstash/current/output-plugins.html
https://www.elastic.co/guide/en/logstash/7.6/input-plugins.html
https://www.elastic.co/guide/en/logstash/7.6/filter-plugins.html
https://www.elastic.co/guide/en/logstash/7.6/output-plugins.html
```

```powershell
# logstash-9.0.1不能以root启动，还要求对/usr/share/logstash/data有权限
[root@logstash01 ~]# chmod 777 /usr/share/logstash/data
[root@logstash01 ~]# su - wang
wang@logstash01:~$ /usr/share/logstash/bin/logstash -e 'input { stdin{} } output { stdout{}}'
```

##### 从文件输入
```powershell
vim file_to_stdout.conf
input {
    file {
        path => "/tmp/wang.*"
        type => "wanglog"             #添加自定义的type字段,可以用于条件判断,和filebeat中tag功能相似
        exclude => "*.txt"            #排除不采集数据的文件，使用通配符glob匹配语法
        start_position => "beginning" #第一次从头开始读取文件,可以取值为:beginning和end,默认为end,即只从最后尾部读取日志
        stat_interval => "3"             #定时检查文件是否更新，默认1s
        codec => json              #如果文件是Json格式,需要指定此项才能解析,如果不是Json格式而添加此行也不会影响结果
    }
}

output {
    stdout {
        codec => rubydebug
    }
}

[root@ubuntu2404 conf.d]#echo "hello" > /tmp/wang.log
[root@ubuntu2404 conf.d]#ls /tmp/wang.log
/tmp/wang.log
[root@ubuntu2404 conf.d]#cat /tmp/wang.log
hello
[root@ubuntu2404 conf.d]#echo "123456789" > /tmp/wang.txt
```

##### 从 Http 请求输入数据
```powershell
cat /etc/logstash/conf.d/http_to_stdout.conf
input {
    http {
        port => 6666
        codec => json
    }
}

output {
    stdout {
        codec => rubydebug
    }
}

[root@logstash ~]#/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/http_to_stdout.conf -r
#执行下面访问可以看到上面信息
[root@ubuntu2004 ~]#curl http://logstash.wang.org:6666
ok
[root@ubuntu2004 ~]# curl -XPOST -d'test log message' http://10.0.0.106:6666
```

##### 从 Filebeat 输入数据
```powershell
[root@ubuntu2404 conf.d]#vim filebeat_to_stdout.conf
input {
  beats {
    port => 5044
  }
}

output {
    stdout {
        codec => rubydebug
    }
}
```

```powershell
vim /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: filestream
  id: my-filestream-id-1
  enabled: true
  tags: ["nginx-access"]     #指定tag,用于分类
  paths:
    - /var/log/nginx/access_json.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选
- type: filestream
  id: my-filestream-id-2
  enabled: true
  tags: ["nginx-error"]
  paths:
    - /var/log/nginx/error.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选

output.logstash:
  hosts: ["10.0.0.106:5044"] #指定Logstash服务器的地址和端口
```

##### 从 Redis 输入数据 
支持由多个 Logstash 从 Redis 读取日志,提高性能

Logstash 从 Redis 收集完数据后,将删除对应的列表Key

```powershell
cat /etc/logstash/conf.d/redis_to_stdout.conf
input {
	redis {
		host => '10.0.0.200'
		port => "6379"
		password => "123456"
		db => "0"
		data_type => 'list'
		key => "m65-tomcat"
	}
}

output {
    stdout {
        codec => rubydebug
    }
}
```

##### 从 Kafka 输入数据
```powershell
# logstash会读取kafka最新生成的消息数据，原有的消息不会读取
[root@logstash ~]# cat /etc/logstash/conf.d/kakfa_to_stdout.conf
input {
    kafka {
        bootstrap_servers => "10.0.0.201:9092,10.0.0.202:9092,10.0.0.203:9092"
        group_id => "logstash" #多个logstash的group_id如果不同，将实现消息共享（发布者/订阅者模式），如果相同（建议使用），则消息独占（生产者/消费者模式）
        #topics => ["nginx-accesslog","nginx-errorlog"]
        topics => ["filebeat-log2"]
        codec => "json"
        #auto_offset_reset="earliest"
        consumer_threads => 8
    }
}

output {
    stdout {
        codec => rubydebug
    }
}
```

```powershell
vim /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: filestream
  id: my-filestream-id-1
  enabled: true
  tags: ["nginx-access"]     #指定tag,用于分类
  paths:
    - /var/log/nginx/access_json.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选
- type: filestream
  id: my-filestream-id-2
  enabled: true
  tags: ["nginx-error"]
  paths:
    - /var/log/nginx/error.log         #注意:文件不能太小,要求超过1024字节
  parsers:
    - ndjson:
        target: ""    #解析结果存放在指定字段下,如果为空,则保存在根下
        #message_key: message   #对哪个字段做json解析,可选
output.kafka:
  hosts: ["10.0.0.201:9092"]
  topic: filebeat-log2       #指定kafka的topic
  partition.round_robin:
    reachable_only: true    #true表示只发布到可用的分区，不可用分区就不发布，false 时表示所有分区，如果一个节点down，会block
  required_acks: 1          #如果为0，不启用确认机制，错误消息可能会丢失，1等待写入主分区（默认），-1等待写入副本分区
  compression: gzip
  max_message_bytes: 1000000 #每条消息最大长度，以字节为单位，如果超过将丢弃


```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765105824780-02095aa9-bb09-4d52-ab6d-2b5ddcb11d20.png)

