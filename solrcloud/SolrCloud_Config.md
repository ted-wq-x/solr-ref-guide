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

------------

## CREATE: Create a Collection

`/admin/collections?action=CREATE&name=name&numShards=number&replicationFactor=number&maxShardsPerNode=number&createNodeSet=nodelist&collection.configName=configname`   
 
### Input
 
查询参数：

|Key|Type|Required|Default|Description|
|----|----|----|-----|-----|
|name|string|yes||被创建集合的名称|
|router.name|string|no|compositeId|路由：定义在分片之间如何分发文档，值为`implicit`或者`compositeId`。`implicit`路由器不会自动将文档路由到不同的分片。即使你在查询时指定分片也没用。'compositeId'路由器根据uniqueKey字段获取hash，根据hash确定在集群中的分片来接受文档。且有手动路由的功能。当使用'implicit'路由器时，需要使用shards参数。使用'compositeId'路由器，需要numShards参数|
|numShards|integer|No|empty|集合的分片数量，使用`compositeId`时必须|
|shards|string|No|empty|以逗号分隔的分片名称列表，例如shard-x，shard-y，shard-z。 这是使用`implicit`路由器时必须的参数|
|replicationFactor|integer|NO|1|每个分片副本的数量|
|maxShardsPerNode|integer|NO|1|在创建集合时，分片和/副本将分布在所有可用（即活动）节点上，同一分片的两个副本将永远不在同一个节点上。如果在调用CREATE操作时节点不活动，则不会获取新集合的任何部分，这可能导致在单个活动节点上创建的副本太多。该参数限制每个节点副本的数量|
|createNodeSet|string|No||运行创建新的节点拓展集合。如果没有提供该参数，则create操作将会为所有active节点创建shard-replica。格式为node_name，多个使用逗号隔开，如localhost:8983_solr,localhost:8984_solr。或者在创建集合是使用empty，然后使用addreplica操作添加分片副本|
|createNodeSet.shuffle|boolean|No|true|控制创建分片副本的顺序由createNodeSet决定，或者在创建单个副本前将节点列表进行洗牌。false表示列表的顺序是可预测的，能够对副本的位置进行精确的控制，但是true能确保副本能够均匀分布的更好。如果createNodeSet没有设置该参数将忽略|
|collection.configName|string|No|empty|collection的配置名称（前提是配置信息先存到zk上），如果没有则默认采用collection名称作为配置名称|
|router.field|string|No|empty|如果指定了此字段，则路由器将查看输入文档中的字段值，以计算散列值，并标识分片，而不是查看uniqueKey字段。如果文档中指定的字段为空，则该文档将被拒绝。请注意，实时获取或通过ID检索还需要参数_route_（或shard.keys）以避免分布式搜索|
|property.name=value|string|No||设置core的属性设置|
|autoAddReplicas|Boolean|No|false|当设置为true时，使得在共享文件系统上自动添加副本|
|async|string|No||请求ID跟踪此操作将被异步处理。|
|rule|string|No||副本放置规则。|
|snitch|string|No||snitch提供者的详细信息|
    

### Output    
    
响应将包括请求的状态和新的核心名称。如果状态是“success”以外的其他状态，则将返回错误消息。 

------------
    
## MODIFYCOLLECTION: Modify Attributes of a Collection

`/admin/collections?action=MODIFYCOLLECTION&collection=<collection-name>&<attributename>=<attribute-value>&<another-attribute-name>=<another-value>`   

一次可以编辑多个属性。更改这些值仅更新ZooKeeper上的z节点，它们不会更改集合的拓扑结构。例如，增加replicationFactor将不会自动为该集合添加更多副本，但将允许更多ADDREPLICA命令成功。也就是之后更改后续的节点。

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|被修改集合名称|
|`<attribute-name>`|string|Yes|键值对，可以被修改的键值对有，maxShardsPerNode、maxShardsPerNode、autoAddReplicas、collection.configName、rule、snitch|

------------

## RELOAD: Reload a Collection

当你修改了zk上的配置后，使用下面的命令修改`/admin/collections?action=RELOAD&name=name`

查询参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|name|string|Yes|被重新加载的集合名称|
|async|string|No|请求ID跟踪此操作将被异步处理。|

## SPLITSHARD: Split a Shard

`/admin/collections?action=SPLITSHARD&collection=name&shard=shardID`

shard拆分将会使得已经存在的shard拆分为2个且数据被写到新的shards上。原始分片将继续包含相同的数据，但它将开始重新路由请求到新的分片。
新的shard将具有与原始shard一样多的副本。分割shard后将自动发起软提交，以便于在子shard上的文档可用。
拆分操作后，不需要显式提交（硬或软），因为在拆分操作期间索引会自动保留到磁盘。    
    
此命令允许无缝拆分，无需停机。被分割的shard能继续接受查询和索引请求，并且一旦完成操作，将自动将数据路由到新的分片上。**此命令只能用于使用带有numShards参数创建的cloud集合**，这意味着集合依赖于solr的hash路由机制。

通过将原始分片的hash范围划分为两个相等的分区并根据新的子范围分割原始分片中的文档。                                                               

还可以指定一个可选范围参数，以将原始分片的散列范围划分为以十六进制表示的任意散列范围间隔。例如，如果原始散列范围为0-1500，则添加参数：ranges = 0-1f4,1f5-3e8,3e9-5dc将原始分片划分为三个碎片，散列范围为0-500,501-1000和1001-1500。    
    
另一个可选参数`split.key`可用于使用路由密钥拆分分片，以使指定路由密钥的所有文档最终在单个专用子分片中。
在这种情况下，不需要提供'shard'参数，因为路由密钥足够找出正确的分片。不支持跨越多个分片的路由密钥 例如，假设split.key = A！ 散列到范围12-15，属于范围0-20的碎片'shard1'，然后通过此路由密钥分割将产生三个子范围为0-11,12-15和16-20的子分片。
注意，具有路由密钥的散列范围的子分片还可以包含其散列范围重叠的其他路由密钥的文档。   
    
由于分割shard时间很长，要使用异步调用避免超时。

查询参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|被分割的集合名称|
|shard|string|yes|被分割的shard名称|
|ranges|string|No|以逗号分隔的十六进制hash范围列表，例如range = 0-1f4,1f5-3e8,3e9-5dc。|
|split.key|string|No|分割索引的key|
|property.name=value|string|No||设置core属性设置，参见core的属性部分|
|async|string|No|请求ID跟踪此操作将被异步处理。|   
    
---------

## CREATESHARD: Create a Shard          
    
使用了"implicit"路由器的集合才能使用此api，使用“compositeId”路由器的集合使用SPLITSHARD。可以为现有的“implicit“集合创建一个名称为新的分片。
                                    
`/admin/collections?action=CREATESHARD&shard=shardName&collection=name`

参数：

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|yes|集合名称|
|shard|string|Yes|被创建shard的名称|
|createNodeSet|string|No|定义能够拓展新集合的节点。如果没有提供，则create操作在所有节点上创建分片副本。格式是以逗号分隔的node_name列表。例如localhost:8983_solr,localhost:8984_solr, localhost:8985_solr|
|property.name=value|string|No||设置core属性设置，参见core的属性部分|
|async|string|No|请求ID跟踪此操作将被异步处理。|   

-------------

## DELETESHARD: Delete a Shard                                                                           
    
删除分片将卸载分片的所有副本，并从clusterstate.json文件中删除它们，并且（默认情况下）删除每个副本的instanceDir和dataDir目录。
注意只会删除处于inactive状态下的分片或者不在自定分片的范围内的。  
    
`/admin/collections?action=DELETESHARD&shard=shardID&collection=name`        
    
参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|yes|集合名称|
|shard|string|Yes|被删除shard的名称|                                                                                        
|deleteInstanceDir|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个instanceDir。将其设置为false以防止实例目录被删除。|                             
|deleteIndex|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个dataDir。将其设置为false以防止数据目录被删除。|     
|deleteIndex|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个索引。将其设置为false以防止索引目录被删除。|     
|async|string|No|请求ID跟踪此操作将被异步处理。|   

--------

## CREATEALIAS: Create or Modify an Alias for a Collection

CREATEALIAS操作将创建一个指向一个或多个集合的新别名。 如果具有相同名称的别名已经存在，则此操作将替换现有别名，有效地表现为原子“MOVE”命令。

`/admin/collections?action=CREATEALIAS&name=name&collections=collectionlist`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|name|string|Yes|别名|
|collection|string|yes|集合名称|    
|async|string|No|请求ID跟踪此操作将被异步处理。|   

----------------

## DELETEALIAS: Delete a Collection Alias

`/admin/collections?action=DELETEALIAS&name=name`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|name|string|Yes|要删除的别名|
|async|string|No|请求ID跟踪此操作将被异步处理。|   

--------------

## LISTALIASES: List of all aliases in the cluster

`/admin/collections?action=LISTALIASES`

------------

## DELETE: Delete a Collection

`/admin/collections?action=DELETE&name=collection`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|name|string|Yes|要删除的集合|
|async|string|No|请求ID跟踪此操作将被异步处理。|   

-------------

## DELETEREPLICA: Delete a Replica

从指定的集合和分片中删除指定的副本。如果对应的core已经启动并运行，core将被卸载，该条目将从clusterstate中删除，并且（默认情况下）删除该instanceDir和dataDir。 
如果节点/内核关闭，则该条目将从集群状态中取出，如果核心稍后重新启动，则会自动注销。

`/admin/collections?action=DELETEREPLICA&collection=collection&shard=shard&replica=replica`
    
参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|yes|集合名称|
|shard|string|Yes|包含被删除备份的shard名称|      
|replica|string|No|被删除备份的名称，如果使用count属性则该属性不是必须的|   
|count|integer|No|被删除备份的数量。如果参数值超过实际值，则不会删除副本。如果只有一个副本，也不会删除，如果使用replica参数，则该属性不是必须的|                                
|deleteInstanceDir|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个instanceDir。将其设置为false以防止实例目录被删除。|                             
|deleteIndex|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个dataDir。将其设置为false以防止数据目录被删除。|     
|deleteIndex|boolean|No|默认情况下，Solr将删除每个被删除的副本的整个索引。将其设置为false以防止索引目录被删除。|     
|onlyIfDown|boolean|No|当设置为“true”时，如果副本处于活动状态，则不会执行任何操作。 默认为false|     
|async|string|No|请求ID跟踪此操作将被异步处理。|   

--------------

## ADDREPLICA: Add Replica

`/admin/collections?action=ADDREPLICA&collection=collection&shard=shard&node=nodeName`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|yes|集合名称|
|shard|string|Yes*|将要添加备份的shard名称，route和shard参数必须有一个| 
|route|string|Yes*|如果确切的分片名称不知道，用户可能会通过路由值，系统会识别分片名称。如果指定shard则会忽略该值|
|node|string|No|创建备份的节点名称|    
|instanceDir|string|No|要创建core的instanceDir|   
|dataDir|string|No|要创建core的dataDir|   
|property.name=value|string|No|core的属性|   
|async|string|No|请求ID跟踪此操作将被异步处理。|   

---------

## CLUSTERPROP: Cluster Properties

添加，修改或删除集群的共同属性

`/admin/collections?action=CLUSTERPROP&name=propertyName&val=propertyValue Input`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|name|string|Yes|属性名称，支持的属性名称有urlScheme，autoAddReplicas，location。其他属性不支持|
|val|string|Yes|属性值，如果只为空或null，则不会设置|

--------

## MIGRATE: Migrate Documents to Another Collection

`/admin/collections?action=MIGRATE&collection=name&split.key=key1!&target.collection=target_collection&forward.timeout=60`

MIGRATE命令用于将具有给定路由密钥的所有文档迁移到另一个集合。源集合将拥有相同的数据，但它开始将请求路由到目标集合，forward.timeout指定超时时间。
在“MIGRATE”命令完成后，用户有责任切换到目标集合进行读写操作。

由`split.key`参数指定路由key，该参数在源和目标集合的分片都是能用的。迁移操作子单线程中以分片的方式进行。
在“迁移”过程中，此命令会创建一个或多个临时集合，但在结束时会自动清除它们。

这个命令的执行时间很长所以推荐使用`async`参数。如果没有指定同步参数，则默认情况下操作是同步的，并且有最大超时时间。
即使设置很大的超时时间，由于集合api的固有限制，依然可能超时，但并不意味着操作失败。用户应在调用操作之前检查日志，集群状态，源和目标集合。

该命令只能在使用'composited'路由器时使用，目标集合在迁移命令运行期间不得接收任何写入，否则有些写入可能会丢失。

请注意，迁移API不会对文档执行任何重复数据删除，因此，如果目标集合包含与正在迁移的文档具有相同uniqueKey的文档，则目标集合将存在重复的文档。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|源集合名称|
|target.collection|string|Yes|目标集合名称|
|split.key|string|Yes|路由规则前缀，例如uniquekey`a!123`，则可能使用`split.key=a!`|
|forward.timeout|int|No|在给定的split.key的源集合写入请求之前，超时时间（秒）将被转发到目标分片。默认值为60秒。|
|property.name=value|string|No|core的属性|   
|async|string|No|请求ID跟踪此操作将被异步处理。| 

----------------------

## ADDROLE: Add a Role

`/admin/collections?action=ADDROLE&role=roleName&node=nodeName`

给集群中某个的节点分配角色。4.7版本中只支持’overseer’。使用此api将某个节点作为”监督者“。多次调用添加更多节点。
监督者对于超大集群很用。如果可用，分配“监督”角色的节点列表中的一个将成为监督者。
如果指定的节点没有启动并运行，系统将将会把角色分配给任何其他节点。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|role|string|Yes|角色名称，目前只支持overseer|
|node|string|Yes|节点的名称。可行的话在该节点启动之前可以分配一个角色|

响应将包括请求的状态和更新或删除的属性。 status=0表示成功。

--------------

## REMOVEROLE: Remove Role

`/admin/collections?action=REMOVEROLE&role=roleName&node=nodeName`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|role|string|Yes|角色名称，目前只支持overseer|
|node|string|Yes|节点的名称|

响应结果中status=0表示成功。

----------

## OVERSEERSTATUS: Overseer Status and Statistics

返回overseer的当前状态，overseer各种API的性能统计信息以及每个操作类型最新的10个故障。

`/admin/collections?action=OVERSEERSTATUS`

-----------

## CLUSTERSTATUS: Cluster Status

获取集群状态，包括集合，分片，副本，配置名称以及集合别名和集群属性。

`/admin/collections?action=CLUSTERSTATUS`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|No|集合名称|
|shard|string|No|指定分片，多个分片使用逗号隔开|
|route|string|No|如果您需要特定文档所在的分片的详细信息，并且您不知道它属于哪个分片，则可以使用它|

-----------

## REQUESTSTATUS: Request Status of an Async Call

请求已经提交的异步收集API（下面）调用的状态和响应。 此接口也用于清除存储的状态。

`/admin/collections?action=REQUESTSTATUS&requestid=request-id`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|requestid|string|Yes|用户定于的id，用于异步追踪|

-----------

## DELETESTATUS: Delete Status

删除已经失败会完成的异步响应

`/admin/collections?action=DELETESTATUS&requestid=request-id`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|requestid|string|No|需要清除的请求id|
|flush|boolean|No|true将清除所有存储的完成或者失败的响应|

-------------

## LIST: List Collections

获取集群中所有集合名称吗，没有参数。

`/admin/collections?action=LIST`

---------------

## ADDREPLICAPROP: Add Replica Property

为特定副本分配一个任意属性并给它指定的值。如果属性已经存在，它将被新值覆盖。

`/admin/collections?action=ADDREPLICAPROP&collection=collectionName&shard=shardName&replica=replicaName&property=propertyName&property.value=value`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|副本所属集合的名称|
|shard|string|Yes|副本所属分片的名称|
|replica|string|Yes|副本名称|
|property|string|Yes|添加的属性，注意：这有个'properties'标签，用于和系统属性区分开，所以下面的两种属性是等价的，`property=special`，`property=property.special`|
|property.value|string|Yes|属性的值|
|shardUnique|Boolean|NO|默认值：false。 如果为true，则该属性设置为一个副本并将从该分片中的所有其他副本中删除该属性。有一个预定义属性preferredLeader，其中shardUnique被强制为“true”，如果shardUnique显式设置为“false”，则返回错误。 PreferredLeader是一个布尔属性，任何不为true（不区分大小写）的值将被解释为preferredLeader的“false”。|

--------------

## DELETEREPLICAPROP: Delete Replica Property

从特定副本中删除任意属性。

`/admin/collections?action=DELETEREPLICAPROP&collection=collectionName&shard=shardName&replica=replicaName&property=propertyName`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|副本所属集合的名称|
|shard|string|Yes|副本所属分片的名称|
|replica|string|Yes|副本名称|
|property|string|Yes|删除的属性，注意：这有个'properties'标签，用于和系统属性区分开，所以下面的两种属性是等价的，`property=special`，`property=property.special`|

--------------

## BALANCESHARDUNIQUE: Balance a Property Across Nodes

`/admin/collections?action=BALANCESHARDUNIQUE&collection=collectionName&property=propertyName`

确保特定属性在构成集合的物理节点之间均匀分布。 如果属性已经存在于副本上，则会尽力将其留在那里。 如果属性不在分片的任何副本上，则选择一个副本并添加该属性。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|属性所属的集合|
|property|string|Yes|平衡属性，如果没有明确指定则添加‘property.’前缀|
|onlyactivenodes|boolean|No|默认为true。即只分发到处于active状态的节点上。 若为false，则同时包含非活动节点。|
|shardUnique|boolean|No|没法翻译，自己理解吧 :sob:|

--------------

## REBALANCELEADERS: Rebalance Leaders

对集合中处于active状态的节点，根据preferredLeader属性重新分配leader。

`/admin/collections?action=REBALANCELEADERS&collection=collectionName`

在使用BALANCESHARDUNIQUE或者ADDREOLICAPROP命令设置preferredLeader属性后。运行上面的命令。
注意：不需要集合中的所有分片都具有preferredLeader属性。重新平衡只是尝试将领导权重新分配到preferredLeader值为true且处于active状态的副本上，
而不是当前分片的leader。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|要重新平衡首选列表的集合的名称|
|maxAtOnce|string|No|一次分配的最大分配数|
|maxWaitSeconds|string|No|默认60s|

TODO：这块没理解，翻译的有问题。

----------

## FORCELEADER: Force Shard Leader

在leader节点由于不幸的事情发生的情况下，这个命令将会强制选取新的leader。

`/admin/collections?action=FORCELEADER&collection=<collectionName>&shard=<shardName>`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|集合的名称|
|shard|string|Yes|分片的名称|

注意：这是一个专家级别的命令，只有在常规leader选举不起作用的时候才应该被调用。如果新的leader确实没有更新，可能是最近的被前leader接受的更新，这可能会导致数据丢失。

------------

## MIGRATESTATEFORMAT: Migrate Cluster State

一个专家级实用程序API，用于将zookeeper节点中共享的clusterstate.json（由stateFormat = 1创建，5.0版本之前的默认属性）中的集合移动到存储在ZooKeeper中的每个集合state
.json（由stateFormat = 2创建），当前默认）无缝地没有任何应用程序停机。

`/admin/collections?action=MIGRATESTATEFORMAT&collection=<collection_name>`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|从clusterstate.json迁移到其自己的state.json zookeeper节点的集合的名称|
|async|string|No|同上|

此API在将Solr 5.0之前创建的任何集合迁移到默认情况下现在使用的可扩展集群状态格式时非常有用。 如果在任何Solr 5.x版本或更高版本中创建了集合，则不需要执行此命令。

-----------

## BACKUP: Backup Collection

备份solr集合，并将其和共享文件系统相关联。

`/admin/collections?action=BACKUP&name=myBackupName&collection=myCollectionName&location=/path/to/my/shared/drive`

备份命令将备份指定集合的Solr索引和配置。备份命令从每个分片获取索引的一个副本。对于配置，它备份与集合和元数据相关联的configSet。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|需要备份的集合名称|
|location|string|No|备份数据存放的位置|
|async|string|No|同上|
|repository|string|No|要用于备份的存储库的名称。如果没有指定存储库，那么本地文件系统存储库将被自动使用|
                      
------------

## RESTORE: Restore Collection

恢复索尔索引和相关配置。

`/admin/collections?action=RESTORE&name=myBackupName&location=/path/to/my/shared/drive&collection=myRestoredCollectionName`
 
恢复操作将使用collection参数值创建一个带有指定名称的集合。您无法将备份恢复到同一个集合中，并且在恢复过程中调用API时，目标集合不存在。

创建的集合将具有与原始集合相同数量的分片和副本，保留路由信息等。可选地，您可以覆盖下面所述的一些参数。
在恢复时，如果ZooKeeper中存在一个具有相同名称的configSet，那么Solr将会重新使用，否则将在ZooKeeper中上传备份的configSet并使用它。

您可以使用集合别名API来确保客户端不需要将端点更改为针对新恢复的集合进行查询或索引。

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|存储恢复数据的集合|
|location|string|No|备份数据存放的位置|
|async|string|No|同上|
|repository|string|No|要用于备份的存储库的名称。如果没有指定存储库，那么本地文件系统存储库将被自动使用|
    
另外有些可被重写的参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection.configName|string|No|empty|collection的配置名称（前提是配置信息先存到zk上），如果没有则默认采用collection名称作为配置名称|
|replicationFactor|integer|NO|1|每个分片副本的数量|
|maxShardsPerNode|integer|NO|1|在创建集合时，分片和/副本将分布在所有可用（即活动）节点上，同一分片的两个副本将永远不在同一个节点上。如果在调用CREATE操作时节点不活动，则不会获取新集合的任何部分，这可能导致在单个活动节点上创建的副本太多。该参数限制每个节点副本的数量|
|autoAddReplicas|Boolean|No|false|当设置为true时，使得在共享文件系统上自动添加副本|
|property.name=value|string|No||设置core的属性设置|
    
--------------

## DELETENODE: Delete Replicas in a Node

删除该节点中所有集合的所有副本。请注意，此操作后，节点本身将保持为活动节点。
``
`/admin/collections?action=DELETENODE&node=nodeName`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|node|string|Yes|被清理的节点|
|async|string|No|同上|

-----------

## REPLACENODE: Move All Replicas in a Node to Another

这个命令将在新的节点上重建源节点的备份。在每个备份都被复制之后，源节点的备份将被删除。

`/admin/collections?action=REPLACENODE&source=source-node&target=target-node`

参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|source|string|Yes|源节点|
|target|string|Yes|目标节点|
|parallel|boolean|No|默认值为false。如果此标志设置为true，则所有副本都将创建为独立线程。如果副本有这种情况，这可能会导致非常高的网络和磁盘I/O非常大。|
|async|string|No|同上|
                    
此操作不会在属于源节点的副本上保留必需的锁。所以在此期间不要执行其他收集操作。

--------------

## MOVEREPLICA: Move a Replica to a New node

此命令将副本从节点移动到新节点，以防共享文件系统，dataDir将被重装。

`/admin/collections?action=MOVEREPLICA&collection=collection&shard=shard&replica=replica&node=nodeName&toNode=nodeName`
 
参数

|Key|Type|Required|Description|
|----|----|-----|-----|
|collection|string|Yes|几个名称|
|shard|string|Yes|副本所属分片名称|
|replica|string|Yes|副本名称|
|node|string|Yes|包含副本的节点名称|
|toNode|string|Yes|目标节点名称|
|async|string|No|同上|    