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