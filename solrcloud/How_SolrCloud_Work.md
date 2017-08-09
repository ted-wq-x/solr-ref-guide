## Key SolrCloud Concepts

SolrCloud集群由一些“逻辑”概念组成，这些概念位于“物理”概念之上。

### Logical

* 群集可以托管Solr文档的多个集合。
* 集合可以被分割成多个碎片，集合包含的是文档的子集
* 集合的分片数量是确定的
    * 集合中合理包含的文档的数量理论上是有限的
    * 一次查询请求可能的并发量

### Physical

* 一个集群由多个solr节点组成，每个solr节点都是一个solr服务进程。
* 每个节点可以托管多个core
* 一个集群中的每个core都是逻辑分片的物理副本
* 每个副本使用与它所属的集合指定的相同配置
* 每个分片的副本的数量是确定的
    * 集合中内置的冗余级别以及群集的容错能力如何某些节点变得不可用。
    * 在高负荷下的并发查询的理论限制

## Shards and Indexing Data in SolrCloud

当你的集合对于一个节点来说太大时，你可以通过创建多个碎片将部分数据分解并存储

分片是集合的逻辑分区，包含集合中文档的子集，使得集合中的每个文档都包含在一个碎片中。
哪个分片包含一个集合中的每个文档完全取决于该集合的“分片”策略。例如，您可能有一个集合，其中每个文档的“country”字段存储在哪一个分片上是确定的，
所以能够集中定位来自同一国家的documents。不同的集合可能简单的使用“hash”作为每个document的uniqueKey以确定自身的碎片位置。

在SolrCloud之前，solr支持分布式查询，它支持一条查询语句跨多个分片，所以查询是针对整个solr索引执行的并且不会出现document被搜索结果遗漏的情况，
因此分片之间分割索引不仅仅是solrCloud的概念。然而，使用SolrCloud需要改进的分布式方法存在若干问题：

1. 将索引分割成碎片有些手动
2. 不支持分布式索引，这意味着您需要显式发送文件到特定的分片; Solr自己找不到被发送文件的碎片
3. 没有负载平衡或故障切换，所以如果你有大量的查询，你需要弄清楚发送请求到哪里，如果一个碎片挂了，请求也就失败了。 
     
SolrCloud解决了所有这些问题。 支持自动分发索引进程和查询，ZooKeeper提供故障切换和负载平衡。 此外，每个分片还可以具有多个副本，以实现更多的鲁棒性。

在SolrCloud中，没有主从的概念。相反，每个分片由至少一个物理副本组成，其中一个是领导者。领导者自动选举，最初以先到先得的方式，然后基于ZooKeeper进程描述
http://zookeeper.apache.org/doc/trunk/recipes.html#sc_leaderElection..                              

如果主节点挂了，则将自动从副本中选取一个作为主节点。

当文档发送到Solr节点进行索引时，系统首先确定该document属于哪个Shard，然后确定哪个节点是该分片的leader。然后将文档转发到当前的leader进行索引，并且leader将更新转发到所有其他副本。

## Document Routing

Solr可以通过在创建集合时，使用router.name参数来指定集合使用的路由器实现。

如果使用（默认）“compositeId”路由器，则可以在发送document时在`document ID`前添加一个前缀该前缀用于计算Solr的hash以确定文档发送到哪个分片进行索引。
前缀可以是任意的（例如，它不一定是分片名称），但它必须一致使得Solr行为一致。例如，如果您想要为一个文件共同定位客户，您可以使用客户名称或ID作为前缀。 如果您的客户是“IBM”，例如
ID为“12345”的文档，您可以将前缀插入到文档ID字段中：“IBM！12345”。该感叹号（'！'）在这里是至关重要的，前缀用于区分文档的分片位置。

compositeId路由器支持包含最多2个路由级别的前缀，例如：首先按地区排列的前缀，然后由客户：“USA！IBM！12345”。

如果客户“IBM”有很多文档，并且您想要跨越多个分片传播。 这种用例的语法是：“shard_key /num！document_id”，其中/num是从组合哈希中使用的分片键中的位数。

所以， "IBM/3!12345"将会从shard key中获取三位，从唯一的文档ID中取出29位，它将跨越1/8的碎片数量传播文档，同样，如果num值为2，那么它将跨越1/4的碎片数量传播文档。
在查询时，可以使用_route_参数（即`q=solr＆_route_=IBM/3！`）将查询的前缀和查询中的位数一起包含在内，以便将查询引导到特定的分片上。

如果您不想影响文档的存储方式，则不需要在其中指定前缀文件ID。

如果在创建集合时定义了“implicit”路由器，则可以另外定义一个router.field参数，以使用每个文档的字段来标识文档所在的分片。但是，如果文档中指定的字段丢失，则文档将被拒绝。您也可以使用_route_参数命名特定的分片。

## Shard Splitting

在cloud模式下创建集合时，将会指定初始化分片的数量。但是，很难确定你所需要的分片数量，特别是当组织需求可能会在一瞬间发生变化时，并且稍后将发现错误的成本很高，包括创建新的core和重建数据索引。

Collection API有能力切割分片。它目前允许将分片分成两部分。现有的碎片保持原样，因此拆分动作有效地将数据的两个副本作为新的碎片。你可以在以后准备好的时候删除旧的分片。

## Ignoring Commits from Client Applications in SolrCloud

在大多数情况下，当以SolrCloud模式运行时，索引客户端应用程序不应发送显式提交请求。相反，您应该使用`openSearcher = false`和`auto 
soft-commits`来配置自动提交，以使最近的更新在搜索请求中可见。这样可以确保在群集中定期进行自动提交。

要让执行客户端应用程序不指定提交策略，应更新所有客户端应用程序。然而，这并不总是可行的，所以Solr提供了`IgnoreCommitOptimizeUpdateProcessorFactory`
，它允许你忽略来自客户端应用程序的显式提交和/或优化请求，而无需重构客户端应用程序代码。

要激活此请求处理器，您需要将以下内容添加到solrconfig.xml中：

```xml
<updateRequestProcessorChain name="ignore-commit-from-client" default="true">
    <processor class="solr.IgnoreCommitOptimizeUpdateProcessorFactory">
        <int name="statusCode">200</int>
    </processor>
    <processor class="solr.LogUpdateProcessorFactory" />
    <processor class="solr.DistributedUpdateProcessorFactory" />
    <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>
```

如上面的示例所示，处理器将返回200到客户端，但会忽略提交/优化请求。请注意，您也需要连接SolrCloud所需的隐式处理器这个定制链代替默认链。

在以下示例中，处理器将引发带有自定义错误的403代码的异常信息：
```xml
<updateRequestProcessorChain name="ignore-commit-from-client" default="true">
    <processor class="solr.IgnoreCommitOptimizeUpdateProcessorFactory">
        <int name="statusCode">403</int>
        <str name="responseMessage">Thou shall not issue a commit!</str>
    </processor>
    <processor class="solr.LogUpdateProcessorFactory" />
    <processor class="solr.DistributedUpdateProcessorFactory" />
    <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>
```

最后，您还可以将其配置为忽略优化，并执行通过
```xml
<updateRequestProcessorChain name="ignore-optimize-only-from-client-403">
    <processor class="solr.IgnoreCommitOptimizeUpdateProcessorFactory">
        <str name="responseMessage">Thou shall not issue an optimize, but commits are OK!</str>
        <bool name="ignoreOptimizeOnly">true</bool>
    </processor>
    <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>
```

## Distributed Requests

当一个Solr节点接收到一个搜索请求时，该请求将被路由到被搜索集合的分片及其副本。

被选择的副本充当聚合器：它为集合中每个分片的随机选择的副本创建内部请求，协调响应，根据需要发出任何后续内部请求（例如，细化面值或请求其他存储字段）并为客户端构建最终响应。

### Limiting Which Shards are Queried

虽然使用SolrCloud的优点之一是可以查询分布在各种分片之间的大型集合，但在某些情况下，您可能只对部分分片感兴趣。您可以选择搜索所有数据或部分数据。

查询所有分片的集合应该看起来很熟悉; SolrCloud甚至没有发挥作用：

`http://localhost:8983/solr/gettingstarted/select?q=*:*`

另外，如果想搜索一个分片，则可以指定分片的逻辑ID，如：

`http://localhost:8983/solr/gettingstarted/select?q=*:*&shards=shard1`

一次搜索指定多个分片也是可以的

`http://localhost:8983/solr/gettingstarted/select?q=*:*&shards=shard1,shard2`

在上面的例子中，分片id被用于挑选一个分片的随机副本。或者，你可以指定分片的副本

`http://localhost:8983/solr/gettingstarted/select?q=*:*&shards=localhost:7574/solr/gettingstarted,
 localhost:8983/solr/gettingstarted`
 
或者，指定分片列表（solr进行负载均衡，选个一个分片），使用`|`符号

`http://localhost:8983/solr/gettingstarted/select?q=*:*&shards=localhost:7574/solr/gettingstarted|
 localhost:7500/solr/gettingstarted`
 
下面的例子，查询2个分片，第一个是shard1的随机副本，第二个是显式分隔线列表中的随机副本：

`http://localhost:8983/solr/gettingstarted/select?q=*:*&shards=shard1,localhost:7574/solr/gettings
 tarted|localhost:7500/solr/gettingstarted`
 
### Configuring the ShardHandlerFactory

你可以在Solr中直接配置分布式搜索中使用的并发和线程池。这允许更精细的粒度控制以达到自己的具体要求。 默认配置有利于吞吐量而不是延迟。 

标准的配置，在solrconfig.xml中

```xml
<requestHandler name="standard" class="solr.SearchHandler" default="true">
    <!-- other params go here -->
    <shardHandler class="HttpShardHandlerFactory">
        <int name="socketTimeOut">1000</int>
        <int name="connTimeOut">5000</int>
    </shardHandler>
</requestHandler>
```

可以配置的参数如下：

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
|socketTimeout|0 (use OS default)|socket最长等待时间，ms|
|connTimeout|0 (use OS default)|socket连接/绑定最长时间，ms|
|maxConnectionsPer Host|20|在分布式搜索中对每个分片进行并发连接的最大数量|
|maxConnections|10000|分布式搜索中最大并发连接数|
|corePoolSize|0|用于协调分布式搜索的线程数保留的最低限制|
|maximumPoolSize|Integer.MAX_VALUE|用于协调分布式搜索的最大线程数|
|maxThreadIdleTime|5 seconds|在减少负载之前，线程被缩小之前等待的时间|
|sizeOfQueue|-1|如果指定，线程池将使用后备队列而不是直接切换缓冲区。 高吞吐量系统将要配置为直接切换（用-1）。希望更好延迟的系统将需要配置合理的队列大小来处理请求中的变化。|
|fairnessPolicy|false|选择jvm的公平政策排队，如果启用分布式搜索将以先到先得的方式处理，但以吞吐量为代价，如果禁用吞吐量将比延迟有利，false->有利于吞吐量，true->有利于低延迟|

### Configuring statsCache (Distributed IDF)

为了计算相关性，需要文件和术语统计。当要进行文档统计计算时，Solr提供了四种实现：

* LocalStatsCache：这仅使用本地术语和文档统计来计算相关性。在分片间统一分配的情况下，这样做效果很好。如果没有配置<statsCache>，则此选项是默认选项。
* ExactStatsCache：此实现使用全局值（跨集合）获取文档频率。
* ExactSharedStatsCache：这与其功能中的确切统计信息缓存完全相同，但全局统计信息将被缓存用于后续请求
* LRUStatsCache：此实现使用LRU缓存来保存在请求之间共享的全局统计信息。             

使用<statsCache>标签来定义
    
`<statsCache class="org.apache.solr.search.stats.ExactStatsCache"/>`           

### Avoiding Distributed Deadlock

每个分片提供顶级查询请求，然后向所有其他分片发出子请求。注意要确保服务HTTP请求的最大线程数大于来自顶级客户端和其他分片的请求的总数量。
如果不满足以上条件可能会导致分布式死锁。

例如，在两个分片的情况下可能会发生死锁，每个分片只有一个线程来服务HTTP请求。两个线程都可以同时接收顶层请求，并相互发送子请求。因为服务没有剩余的线程处理请求，
所以传入的请求将被阻塞直到有可用线程，由于它俩都等待子请求完成，所以它俩都会阻塞，造成死锁。确保solr的线程配置正确，避免死锁。

### Prefer Local Shards（性能优化的一个点）

在查询时，solr可以传入一个可选的boolean参数`preferLocalShards`，表示分布式查询优先使用哪个分片的副本。换句话说，如果查询包含
`preferLocalShards = true`，则查询控制器将寻找本地副本来执行查询，而不是从群集中随机选择副本。当查询请求每个文档要求返回多个字段或大字段时，
这是有用的，因为它可以避免在本地可用时通过网络传输大量数据。 此外，此功能可用于最小化具有性能问题的副本造成的影响，因为它可降低这种副本将被其他健康副本命中的可能性。

最后，由于查询控制器必须将查询转发到到大多数分片的非本地副本，所以该功能的值随着集合中分片数的增加而减少。换句话说，此功能对于优化具有少量
碎片和许多副本的集合的查询最为有用。另外，只有当使用Solr的CloudSolrClient才能对所有要查询的集合的副本进行负载均衡请求时，才应该使用此选项。
如果不是负载均衡，则此功能可能在集群中引入热点，因为查询将不会在集群中均匀分布。

------

## Read and Write Side Fault Tolerance

### 读侧的高可用

在SolrCloud集群中，每个单独的节点对读取请求进行负载均衡集合中的所有副本。这段就不翻译了，没啥内容，还有点难懂。
使用solrj客户端的CloudSolrClient，它提供负载均衡。

即使集群中的某些节点处于脱机状态或无法访问，只要可以与每个分片的至少一个副本进行通信，或者每个相关分片的一个副本，
如果用户能够正常响应搜索请求，则Solr节点将能够正确响应，通过碎片或_route_参数限制搜索。每个分片的副本越多，
在发生节点故障的情况下，Solr集群就越有可能处理搜索结果。

zkConnected

只要它可以与它所知道的每个分片的至少一个副本通信，即使在收到请求时它不能与ZooKeeper通信，Solr节点将返回搜索请求的结果。
这通常是从容错的角度来看的首选行为，但如果收集结构发生了重大变化，则节点尚未通过ZooKeeper通知，
则可能会导致陈旧或不正确的结果（即分片可能已被添加或删除 ，或分裂成子碎片）。

每个搜索响应中都包含一个zkConnected标题，指示处理该请求的节点是否与ZooKeeper连接在一起：

Solr Response with partialResults
```json
{
  "responseHeader": {
    "status": 0,
    "zkConnected": true,
    "QTime": 20,
    "params": {
      "q": "*:*"
    }
  },
  "response": {
    "numFound": 107,
    "start": 0,
    "docs": [
      "..."
    ]
  }
}
```

shards.tolerant

如果查询的一个或多个分片完全不可用，那么Solr的默认行为是失败请求。 然而，有很多用例，部分结果是可以接受的，
因此Solr提供了一个布尔的shards.tolerant参数（默认为false）。

如果`shards.tolerant = true`，则可能会返回部分结果。 如果返回的响应不包含所有适当的分片的结果，
那么响应头包含一个称为partialResults的特殊标志.

客户端可以指定“shards.info”以及shards.tolerant参数来检索更细致的细节。

partialResults=true的例子

```json
{
  "responseHeader": {
    "status": 0,
    "zkConnected": true,
    "partialResults": true,
    "QTime": 20,
    "params": {
      "q": "*:*"
    }
  },
  "response": {
    "numFound": 77,
    "start": 0,
    "docs": [ "..." ]
  }
}
```

## Write Side Fault Tolerance

solrCloud通过文档备份实现数据冗余，并且你能够将更新请求发送到集群中的任何节点。
该节点将确定它是否托管适当分片的领导者，如果不是，它将将请求转发给领导者，然后由leader转发到所有现有副本，
使用版本控制实现每个副本的数据都是最新的。如果leader挂了，其他备份将替代它的位置。这种架构能够确保在发生灾难时可以恢复数据，即使使用的是近实时搜索。

### Recovery

为每个节点创建事物日志，记录内容和结构的所有更改。日志被用于确定节点的那些内容应该被备份到备份节点当中。
当创建一个新的备份节点，该节点将和leader通信并通过事物日志确定备份内容。失败，将会重试。

由于事务日志由更新记录组成，因此它可以进行更强大的索引，如果建立索引的过程被中断，则重新设置未提交的更新。

如果leader挂了，它可能已经向某些副本而不是其他副本发送请求。所以如果新的leader被选举出来，它会和其他副本进行同步。
如果成功，所有数据都是一致的，leader被设置为激活状态，其他依旧。如果副本太远，则系统会要求进行完全复制。

如果core正在加载schema或者有未完成的将会导致更新失败，leader将会通知节点更新失败并情动恢复过程。

### Achieved Replication Factor

当使用大于1的复制因子时，分片的leader上的更新请求可能会成功，但在一个或多个副本上会失败。
例如，考虑一个分片，复制因子为3的情况下，你有一个分片的leader和两个额外的副本。 
如果更新请求成功执行，但两个副本都失败，无论什么原因，从客户端的角度来看，更新请求仍然被认为是成功的。
错过更新的副本将与leader在恢复时进行同步。

在幕后，这意味着Solr已经接受了仅在其中一个节点（当前leader）上的更新。Solr支持更新请求上的可选min_rf参数,
该参数使得更新的返回结果中带有复制因子。对于上述示例场景，如果客户端应用程序包含`min_rf> = 1`，
则在响应头中返回`rf = 1`，因为该请求只能在leader上成功执行。
更新请求仍将被接受，因为min_rf参数仅告诉Solr客户端应用程序希望知道实现的更新请求的复制因子。
换句话说，min_rf不意味着Solr将会执行最小的复制因子，因为Solr不支持在副本子集上成功执行回滚更新。
