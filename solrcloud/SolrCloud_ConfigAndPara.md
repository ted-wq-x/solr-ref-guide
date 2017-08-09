# Setting Up an External ZooKeeper Ensemble

使用solr自带的zookeeper是不推荐的，存在操作风险。

How Many ZooKeepers?
> 根据zookeeper官方文档，设F为运行出现故障的最大机器数量，则需要部署zookeeper的机器的数量是2*F+1,另外ZooKeeper的部署通常由奇数机器组成。

## Setting Up a Single ZooKeeper

1. 配置文件，创建`<ZOOKEEPER_HOME>/conf/zoo.cfg`，并在该文件中添加以下内容

```properties
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```

参数说明：
* tickTime：zk集群确定哪些服务器在任意给定的时间内启动并运行，最小会话超时被定义为两个“ticks”。tickTime参数以毫秒为单位。
* dataDir：这是ZooKeeper将存储有关集群的数据的目录。该目录应该开始为空。
* clientPort：Solr将访问ZooKeeper的端口。

只有上述的配置文件配置完成，才能启动zookeeper。

2. 运行zk，执行ZOOKEEPER_HOME/bin/zkServer.sh脚本`zkServer.sh start`。

注：zookeeper的功能很强大，如果需要深入学习的话还是得看官方文档。

3. 在solr上指定zookeeper参数，使用`-z`参数

使用2181端口上的zk，`bin/solr start -e cloud -z localhost:2181 -noprompt`。

添加一个solr节点到zk上，`bin/solr start -cloud -s <path to solr home for new node> -p 8987 -z localhost:2181`。

注意：当你不是使用solr的例子启动solr时，需要在创建集合之前上传配置文件到zk上。

4. 关闭，`zkServer.sh stop`。

## Setting up a ZooKeeper Ensemble

1. 配置文件，和上面的有些不同,zoo.cfg如下
```properties
dataDir=/var/lib/zookeeperdata/1
clientPort=2181
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```
新的参数说明：

* initLimit：允许子节点连接或同步leader的最长时间，以ticks为计算单位，例如：有5个ticks，每个有2000ms，则总的等待时间就是10s。
* syncLimit：关注者和zk同步的时间，以ticks为计算单位。如果关注着和leader的差距太大，则会被舍弃。
* server.X：大白话就是，X的值得和<dataDir>/myid文件中的值一致（该文件得新建），每个zk都必须有，这是zk之间通信端口。X的值在1到255之间。

在第二个zk上的配置文件：
```properties
tickTime=2000
dataDir=c:/sw/zookeeperdata/2
clientPort=2182
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

在第三个zk上的配置文件：
```properties
tickTime=2000
dataDir=c:/sw/zookeeperdata/3
clientPort=2183
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

2. 启动zk，zoo.cfg文件名无所谓，但注意后缀。
```
cd <ZOOKEEPER_HOME>
bin/zkServer.sh start zoo.cfg
bin/zkServer.sh start zoo2.cfg
bin/zkServer.sh start zoo3.cfg
```

3. 将solr添加为关注者`bin/solr start -e cloud -z localhost:2181,localhost:2182,localhost:2183 -noprompt`

关于zk的配置可以参考[xiong_wq](../xiong_wq/README.md)

# Using ZooKeeper to Manage Configuration Files

将文件上传到zk有以下三种方式

* 当您使用bin / solr脚本启动SolrCloud示例时。
* 使用bin / solr脚本创建集合时。
* 将配置集显式上传到ZooKeeper。

## 启动脚本

当您首次使用`bin /solr -e`尝试SolrCloud时，相关的配置集将自动上传到ZooKeeper，并与新创建的集合相关联。

以下命令启动solr，将会创建默认集合名称（gettingstarted）和默认配置集（data_driven_schema_configs）并上传到zk。
`bin/solr -e cloud -noprompt`

您还可以使用bin /solr脚本创建集合时指定上传配置目录，使用`-d`参数

`bin/solr create -c mycollection -d data_driven_schema_configs`

这个脚本将会上传配置文件目录到zk的/configs/mycollection目录下。

## Uploading Configuration Files using bin/solr or SolrJ

在生产环境中，将会使用solr的控制脚本或者solj中的`CloudSolrClient.uploadConfig`方法。

`bin/solr zk upconfig -n <name for configset> -d <path to directory with configset>`

**提示：** 使用版本控制工具管理配置文件。

## Managing Your SolrCloud Configuration Files

更新cloud中的配置文件：
1. 从zk中下载配置文件，使用版本控制工具查看这个过程
2. 做出更改
3. 提交更改
4. push到zk
5. 让zk重新载入集合的配置信息

## Preparing ZooKeeper before first cluster start

如果你将与其他应用程序共享相同的ZooKeeper实例，则应在ZooKeeper中使用chroot，具体看文档。

有一些包含集群配置的固定配置文件，由于其中一些对于集群正常运行至关重要，因此可能需要在首次启动Solr群集之前将此类文件上传到ZooKeeper。
这样的配置文件（不详尽）的示例是solr.xml，security.json和clusterprops.json。

例如，如果你希望将solr.xml保存在ZooKeeper中，以避免将其复制到每个节点的solr_home目录中，可以使用bin /solr命令将其推送到ZooKeeper：

`bin/solr cp file:local/file/path/to/solr.xml zk:/solr.xml -z localhost:2181`

# ZooKeeper Access Control

zk的访问控制，即`ZooKeeper ACLs`。SolrCloud使用ZooKeeper进行共享信息和协调。默认情况下solr不使用ACL。

任何人都能够访问zk中的配置信息是很不安全的，所以需要使用`zookeeper ACL`。

其中zk的配置信息存储在硬盘和（部分）内存当中，内存中的访问控制，由zk内部完成。但是这些内容依然可以通过zk api访问。
外部进程可以连接到ZooKeeper并创建/更新/删除/读取内容; 例如，SolrCloud集群中的Solr节点想要创建/更新/删除/读取，并且SolrJ客户端希望从集群读取。
外部进程负责创建/更新内容并设置ACL，ACL描述谁被允许读取，更新，删除，创建等。ZooKeeper中的每条信息（znode / content）都有自己的一组ACL
且是不能被继承或共享的。Solr中的默认行为是在其创建的所有内容上添加一个ACL。一个允许任何人执行任何操作的ACL（在ZooKeeper术语中称为“不安全的ACL”）。

## How to Enable ACLs

TODO

关于这一块的内容，后面翻译，因为我目前还处于学习阶段，这个先缓一缓，毕竟很多。 :see_no_evil:

# Collections API                

`Collections API`用于使您能够创建，删除或重新加载集合，但在SolrCloud中，您还可以使用它来创建具有特定数量的分片和副本的集合.

## CREATE: Create a Collection

`/admin/collections?action=CREATE&name=name&numShards=number&replicationFactor=number&maxShar
 dsPerNode=number&createNodeSet=nodelist&collection.configName=configname`   
 
查询参数：

|Key|Type|Required|Default|Description|
|----|----|----|-----|-----|
|name|string|yes||被创建集合的名称|
|router.name|string|no|compositeId|路由：定义在分片之间如何分发文档，值为`implicit`或者`compositeId`。
`implicit`路由器不会自动将文档路由到不同的分片。即使你在查询时指定分片也没用。|




     
