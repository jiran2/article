# 配置Linux机器

### 配置系统环境

```
配置文件 /etc/sysctl.conf
# 禁用内存与硬盘交换
vm.swappiness=1
# 设置虚拟内存大小
vm.max_map_count=262144
配置文件：/etc/security/limits.conf
# 进程线程数
* soft nproc 131072
* hard nproc 131072
# 文件句柄数
* soft nofile 131072
* hard nofile 131072
# 内存锁定交换
* soft memlock unlimited
* hard memlock unlimited
```

### 创建ES专用账号

Elasticsearch不能用root账号启动，会报错

```
# 创建 ES 账号，如 elastic
useradd elastic
# 授权 ES 程序目录 elastic 账号权限
# 假设 ES 程序目录、数据目录、日志目录都 在/gpes 目录下
chown -R elastic:elastic /usr/local/soft/elk/*
```

### 配置JDK14+

```java
# ES 最新版本自带 jdk 版本，默认可以不需要配置，建议配置，便于安装其它 java 程序辅助
export JAVA_HOME=/usr/local/soft/jdk-15.0.1
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH
```

# 配置单点

## Elasticsearch配置

### 配置文件

- {ES_HOME}/config/elasticsearch.yml

  ```
  # 集群名称，默认可以不修改，此处 es-single-9200
  cluster.name: es-single
  # 节点名称，必须修改 ，默认修改为当前机器名称，若是多实例则需要区分
  node.name: es-single-9200
  # IP 地址，默认是 local，仅限本机访问，外网不可访问，设置 0.0.0.0 通用做法
  # network.host:192.168.222.100
  network.host:0.0.0.0
  # 访问端口，默认 9200，9300，建议明确指定
  http.port: 9200
  transport.port: 9300
  # 数据目录与日志目录，默认在当前运行程序下，生产环境需要指定
  # path.data: /path/to/data
  # path.logs: /path/to/logs
  # 内存交换锁定，此处需要操作系统设置才生效
  bootstrap.memory_lock: true
  # 防止批量删除索引
  action.destructive_requires_name: true
  # 设置处理器数量，默认无需设置，单机器多实例需要设置
  node.processors: 4
  # 集群发现配置
  # discovery.seed_hosts: ["192.168.222.100:9300"]
  cluster.initial_master_nodes: ["192.168.222.100:9300"]
  ```

- {ES_HOME}/config/jvm.options

  ```
  # 内存堆栈大小，不能超过 1/2 系统内存，多实例要谨慎
  -Xms1g
  -Xmx1g
  # 垃圾回收器 CMS 与 G1，当前 CMS 依然最好
  8-13:-XX:+UseConcMarkSweepGC
  14-:-XX:+UseG1GC
  # GC.log 目录，便于排查 gc 问题，生产需要修改路径指向
  8:-Xloggc:logs/gc.log
  #
  ```

### 启动方式

```shell
# 当前窗口启动，窗口关闭，ES 进程也关闭
.{ES_HOME}/bin/elasticsearch
# 后台进程启动
.{ES_HOME}/bin/elasticsearch -d
```

## Kibana配置

### 配置文件

- {KIBANA_HOME}/config/kibana.yml

  ```shell
  # 访问端口，默认无需修改
  server.port: 5601
  # 访问地址 IP，默认本地
  server.host: "192.168.222.100" 
  # ES 服务指向，集群下配置多个
  elasticsearch.hosts: ["http://192.168.222.100:9200"]
  # Kibana 元数据存储索引名字，默认.kibana 无需修改
  kibana.index: ".kibana_single"
  ```

### 启动方式

```
# 当前窗口内启动
.{KIBANA_HOME}/bin/kibana
# 后台进程启动
nohup .{KIBANA_HOME}/bin/kibana &
# 查看日志
tail -fn 200 .{KIBANA_HOME}/bin/nohup.out
# 地址
http://192.168.222.100:5601
```



# 配置集群

Elastic 集群模式必须至少 2 个实例以上，一般建议 3 个节点以上，保障其中一个节点失效，集群仍然可以服务。

集群模式与单实例模式大部分配置上一样的，仅需修改集群通信差异部分。

| 服务器ip:port        | 节点名称        | JVM        |
| -------------------- | --------------- | ---------- |
| 192.168.222.100:9201 | es-cluster-9201 | jdk-15.0.1 |
| 192.168.222.100:9202 | es-cluster-9202 | jdk-15.0.1 |
| 192.168.222.100:9203 | es-cluster-9203 | jdk-15.0.1 |

## Elasticsearch配置

### 配置文件

- {ES_HOME}/config/elasticsearch.yml

  ```shell
  # 集群名称，默认可以不修改，此处 es-cluster
  cluster.name: es-cluster
  # 节点名称
  node.name: es-cluster-9201
  # IP 地址，默认是 local，仅限本机访问，外网不可访问，设置 0.0.0.0 通用做法
  # network.host: 192.168.222.101
  network.host: 0.0.0.0
  # 网络端口 
  # 依次每台机器设置为 
  # http.port: 9202
  # transport.port: 9302
  # http.port: 9203
  # transport.port: 9303
  http.port: 9201
  transport.port: 9301
  # 集群节点之间指向
  discovery.seed_hosts: ["192.168.222.100:9301", "192.168.222.100:9302","192.168.222.100:9303"]
  cluster.initial_master_nodes:["192.168.222.100:9301","192.168.222.100:9302","192.168.222.100:9303"]
  ```

### 启动方式

```
# 当前窗口启动，窗口关闭，ES 进程也关闭
.{ES_HOME}/bin/elasticsearch
# 后台进程启动
.{ES_HOME}/bin/elasticsearch -d
# 查看节点启动成功没
http://192.168.222.100:9201/_cat/health
```

## Kibana配置

### 配置文件

- {KIBANA_HOME}/config/kibana.yml

  ```shell
  # 访问端口
  server.port: 5602
  # 访问地址 IP，默认本地
  # server.host: "192.168.222.100" 
  server.host: "0.0.0.0" 
  
  # ES 服务指向，集群下配置多个
  elasticsearch.hosts:
  ["http://192.168.222.100:9201","http://192.168.222.100:9202","http://192.168.222.100:9203"]
  # Kibana 元数据存储索引名字，默认.kibana 无需修改
  kibana.index: ".kibana_cluster"
  ```

### 启动方式

```
# 当前窗口内启动
.{KIBANA_HOME}/bin/kibana
# 后台进程启动
nohup .{KIBANA_HOME}/bin/kibana &
# 查看日志
tail -fn 200 .{KIBANA_HOME}/bin/nohup.out
# 地址
http://192.168.222.100:5602
```

### 效果展示

#### 图片展示

可以看到三个节点集群启动成功

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201117232119687.png" alt="image-20201117232119687" style="zoom:50%;" />

#### 命令使用

使用elasticsearch命令查看集群状态

http://192.168.222.100:9203/_cat/

以下命令都可以使用

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201118000248675.png" alt="image-20201118000248675" style="zoom:100%;" align="left"/>

举例：http://192.168.222.100:9203/_cat/nodes

查看当前集群得所有节点信息

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201117235756750.png" alt="image-20201117235756750" style="zoom:100%;" align="left"/>

