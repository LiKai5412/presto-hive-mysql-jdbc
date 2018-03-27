1.搭建前环境准备
CentOS 6.7
java8
Python3.4.4
hadoop2.6.4

2.集群计划
hd1(192.168.174.131) :调度节点（coordinator）
hd2(192.168.174.132):worker节点
hd3(192.168.174.133):worker节点

3.连接器
Presto支持从以下版本的Hadoop中读取Hive数据：支持以下文件类型：Text, SequenceFile, RCFile, ORC
Apache Hadoop 1.x  （hive-hadoop1）
Apache Hadoop 2.x  （hive-hadoop2）
Cloudera CDH 4       （hive-cdh4）
Cloudera CDH 5       （hive-cdh5）
此外，需要有远程的Hive元数据。 不支持本地或嵌入模式。 Presto不使用MapReduce，只需要HDFS

4.单机安装步骤
下载 presto-server-0.100, ( 下载地址：https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.100/presto-server-0.100.tar.gz）或者：链接：http://pan.baidu.com/s/1qYTvTwg 密码：4xz6
将 presto-server-0.100.tar.gz 上传至linux主机（hd1），解压后的文件目录结构如下（称为安装目录）：Presto需要一个用于存储日志、本地元数据等的数据目录。 建议在安装目录的外面创建一个数据目录。这样方便Presto进行升级，如：/presto/data

5.配置文件

在安装目录中创建一个etc目录， 在这个etc目录中放入以下配置文件：
1. config.properties：Presto 服务配置
2. node.properties：环境变量配置，每个节点特定配置
3. jvm.config：Java虚拟机的命令行选项
4. log.properties: 允许你根据不同的日志结构设置不同的日志级别
5. catalog目录：每个连接者配置（data sources）
config.properties
包含了Presto server的所有配置信息。 每个Presto server既是一个coordinator也是一个worker。 但是在大型集群中，处于性能考虑，建议单独用一台机器作为 coordinator，一个coordinator的etc/config.properties应该至少包含以下信息：
123456 coordinator=truenode-scheduler.include-coordinator=falsehttp-server.http.port=18080task.max-memory=1GBdiscovery-server.enabled=truediscovery.uri=http://192.168.174.131:18080
1. coordinator：指定是否运维Presto实例作为一个coordinator(接收来自客户端的查询情切管理每个查询的执行过程)
2. node-scheduler.include-coordinator：是否允许在coordinator服务中进行调度工作, 对于大型的集群，在一个节点上的Presto server即作为coordinator又作为worke将会降低查询性能。因为如果一个服务器作为worker使用，那么大部分的资源都会被worker占用，那么就不会有足够的资源进行关键任务调度、管理和监控查询执行
3. http-server.http.port：指定HTTP server的端口。Presto 使用 HTTP进行内部和外部的所有通讯
4. task.max-memory=1GB：一个单独的任务使用的最大内存 (一个查询计划的某个执行部分会在一个特定的节点上执行)。 这个配置参数限制的GROUP BY语句中的Group的数目、JOIN关联中的右关联表的大小、ORDER BY语句中的行数和一个窗口函数中处理的行数。 该参数应该根据并发查询的数量和查询的复杂度进行调整。如果该参数设置的太低，很多查询将不能执行；但是如果设置的太高将会导致JVM把内存耗光
5. discovery-server.enabled：Presto 通过Discovery 服务来找到集群中所有的节点。为了能够找到集群中所有的节点，每一个Presto实例都会在启动的时候将自己注册到discovery服务。Presto为了简化部署，并且也不想再增加一个新的服务进程，Presto coordinator 可以运行一个内嵌在coordinator 里面的Discovery 服务。这个内嵌的Discovery 服务和Presto共享HTTP server并且使用同样的端口
6. discovery.uri：Discovery server的URI。由于启用了Presto coordinator内嵌的Discovery 服务，因此这个uri就是Presto coordinator的uri。注意：这个URI一定不能以“/“结尾

node.properties
包含针对于每个节点的特定的配置信息。一个节点就是在一台机器上安装的Presto实例，etc/node.properties配置文件至少包含如下配置信息
123 node.environment=testnode.id=bigdata_node_worker_hd1node.data-dir=/presto/data
node.environment： 集群名称, 所有在同一个集群中的Presto节点必须拥有相同的集群名称
node.id： 每个Presto节点的唯一标示。每个节点的node.id都必须是唯一的。在Presto进行重启或者升级过程中每个节点的node.id必须保持不变。如果在一个节点上安装多个Presto实例（例如：在同一台机器上安装多个Presto节点），那么每个Presto节点必须拥有唯一的node.id
node.data-dir： 数据存储目录的位置（操作系统上的路径）, Presto将会把日志和数据存储在这个目录下
jvm.config
包含一系列在启动JVM的时候需要使用的命令行选项。这份配置文件的格式是：一系列的选项，每行配置一个单独的选项。由于这些选项不在shell命令中使用。 因此即使将每个选项通过空格或者其他的分隔符分开，java程序也不会将这些选项分开，而是作为一个命令行选项处理,信息如下：
123456789 -server-Xmx16G-XX:+UseConcMarkSweepGC-XX:+ExplicitGCInvokesConcurrent-XX:+CMSClassUnloadingEnabled-XX:+AggressiveOpts-XX:+HeapDumpOnOutOfMemoryError-XX:OnOutOfMemoryError=kill -9 %p-XX:ReservedCodeCacheSize=150M
log.properties
这个配置文件中允许你根据不同的日志结构设置不同的日志级别。每个logger都有一个名字（通常是使用logger的类的全标示类名）. Loggers通过名字中的“.“来表示层级和集成关系，信息如下：
1 com.facebook.presto=DEBUG
配置日志等级，类似于log4j。四个等级：DEBUG,INFO,WARN,ERROR

Catalog Properties
通过在etc/catalog目录下创建catalog属性文件来完成catalogs的注册。 例如：可以先创建一个etc/catalog/jmx.properties文件，文件中的内容如下，完成在jmxcatalog上挂载一个jmxconnector
1 connector.name=jmx
在etc/catalog目录下创建hive.properties，信息如下：
1234 connector.name=hive-hadoop2hive.metastore.uri=thrift://192.169.168.131:9083hive.config.resources=/root/apps/hadoop/etc/hadoop/core-site.xml,/root/apps/hadoop/etc/hadoop/hdfs-site.xmlhive.allow-drop-table=true
以上，是单机部署presto, 至此已经完成。

6.集群安装步骤
将hd1中的presto-server-0.100拷贝到hd2,hd3上
12 scp -r /root/apps/presto-server-0.100 root@hd2:/root/apps/scp -r /root/apps/presto-server-0.100 root@hd3:/root/apps/
修改hd2中的配置文件：
config.properties
12345 coordinator=falsehttp-server.http.port=18080task.max-memory=1GBdiscovery-server.enabled=truediscovery.uri=http://192.168.174.131:18080
node.properties
123 node.environment=testnode.id=bigdata_node_worker_hd2node.data-dir=presto/data
修改hd3中的配置文件
config.properties
12345 coordinator=falsehttp-server.http.port=18080task.max-memory=1GBdiscovery-server.enabled=truediscovery.uri=http://192.168.174.131:18080
node.properties
123 node.environment=testnode.id=bigdata_node_worker_hd3node.data-dir=presto/data
到此，presto集群配置完毕。

7.运行presto
在hd1,hd2,hd3的presto-server-0.100/bin目录下依次启动presto:
1 ./launcher start
在Presto可以使用如下命令作为一个后台进程启动：
1 bin/launcher start
或者在前台运行, 可查看具体的日志
1 bin/launcher run
停止服务进程命令
1 bin/laucher stop
查看服务进程命令
1 bin/laucher status
查看进程： ps -aux|grep PrestoServer  或 jps


也可通过浏览器界面查看：http://192.168.174.131:18080



8.整合hive测试
想要查询连接到hive中查询数据还需要先启动hive的metastore
启动方式：
1234 bin/hive --service metastore #或者后台启动：bin/hive --service metastore 2>&1 >> /var/log.log &#后台启动,关闭shell连接依然存在:nohup bin/hive --service metastore 2>&1 >> /var/log.log &
如果启动失败，查看hive-site.xml中是否有metastore的如下配置，若没有，加上这段后再启动metasotre.
1234 <property><name>hive.metastore.uris<name><value>thrift://192.168.174.131:9083<value>property>
然后下载 presto-cli-0.100-executable.jar：Presto CLI为用户提供了一个用于查询的可交互终端窗口。CLI是一个 可执行 JAR文件, 这也就意味着你可以像UNIX终端窗口一样来使用CLI ，https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.100/presto-cli-0.100-executable.jar文件下载后，重名名为 presto ， 使用 chmod +x 命令设置可执行权限，执行命令：
下面命令的ip和端口和config.properties中的一致
1 ./presto --server 192.168.174.131:18080 --catalog hive --schema default --debug
退出命令：quit或者exit

9.整合mysql测试
和hive类似，在hd1的etc/目录下新建文件：mysql.properties文件
1234 connector.name=mysqlconnection-url=jdbc:mysql://192.168.174.131:3306connection-user=rootconnection-password=123456
然后将mysql.properties分贝拷贝到hd2和hd3的/etc目录下，重新启动PrestoServer服务。
连接测试：
1 ./presto --server localhost:18080 --catalog mysql --schema test --debug
常用写法：
123 SHOW SCHEMAS FROM mysql;#查询数据库列表SHOW TABLES FROM mysql.test;#查询指定数据库下的数据表SELECT * FROM mysql.test.user;查询指定数据表数据
<dependency><groupId>com.facebook.presto<groupId><artifactId>presto-jdbc<artifactId><version>0.100<version><dependency>
