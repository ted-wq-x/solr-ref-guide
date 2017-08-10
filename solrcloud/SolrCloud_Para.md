# Parameter Reference

## Cluster Parameters

|----|-----|-----|
|numShards|默认值为1|存储hash文档的分片数目。每个分片必须有一个领导，每个领导可以有N个副本|

## SolrCloud Instance Parameters

这些设置在solr.xml中，但默认情况下，host和hostContext参数也可以设置系统属性。

|----|-----|-----|
|host|默认值为本机地址|如果自动查找的地址不正确，使用该参数覆盖|
|hostPort|默认端口为使用`bin/solr -p <port>`指定或者为8983|使用命令指定就行了，别玩花的|
|hostContext|solr默认|solr的web应用访问地址|

## SolrCloud Instance ZooKeeper Parameters

|----|-----|-----|
|zkRun|默认为`localhost:<hostPort+1000>`||
|zkHost|没有默认值|ZooKeeper的主机地址。通常，这是一个逗号分隔的地址列表|
|zkClientTimeout|默认值为15000|客户端和ZooKeeper通话的时间超时时间。|

zkRun和zkHost使用系统属性设置。 默认情况下，zkClientTimeout在solr.xml中设置，但也可以使用系统属性进行设置。

## SolrCloud Core Parameters

|----|-----|-----|
|shard|默认为根据numShards自动分配|指定分片的副本在那个core中|

可以在每个核心的core.properties中指定分片

## Command Line Utilities

使用zk的命令行工具，允许和zk交互。solr的命令行提供了`zkCli.sh`脚本的很多功能。并且使用solr的zkCli和zk的zkCli有些不同，
前者的部分脚本参数被修改了，同时zk的脚本更加的完善。

具体参数，自己用命令的help查看。

### ZooKeeper CLI Examples

这个不翻译，自己看吧！

## SolrCloud with Legacy Configuration Files

如果您从非SolrCloud环境迁移到SolrCloud，则此信息可能会有所帮助。

solr的例子的配置信息很完善，只需要修改下的配置。

1. 在schema.xml, 定义`_version_ `字段

`<field name="_version_" type="long" indexed="true" stored="true" multiValued="false"/>`

2. 在solrconfig.xml，定义`UpdateLog`

```properties
<updateHandler>
    ...
    <updateLog>
        <str name="dir">${solr.data.dir:}</str>
    </updateLog>
    ...
</updateHandler
```

3. `DistributedUpdateProcessor`是默认的更新链的一部分，并自动注入到任何自定义更新链中，因此您实际上不需要对此功能进行任何更改。
但是，如果您希望显式添加它，您仍然可以将其作为updateRequestProcessorChain的一部分添加到solrconfig.xml文件中。 例如：

```properties
<updateRequestProcessorChain name="sample">
    <processor class="solr.LogUpdateProcessorFactory" />
    <processor class="solr.DistributedUpdateProcessorFactory"/>
    <processor class="my.package.UpdateFactory"/>
    <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>
```

如果您不希望将`DistributedUpdateProcessFactory`自动注入到链中（例如，如果要使用SolrCloud功能，但您希望自己分发更新），
请在链中指定`NoOpDistributingUpdateProcessorFactory`更新处理器工厂：

```properties
<updateRequestProcessorChain name="sample">
    <processor class="solr.LogUpdateProcessorFactory" />
    <processor class="solr.NoOpDistributingUpdateProcessorFactory"/>
    <processor class="my.package.MyDistributedUpdateFactory"/>
    <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>
```

在更新处理时，Solr跳过已经在其他节点上运行的更新处理器。

## ConfigSets API

通过ConfigSets API，您可以创建，删除和以其他方式管理ConfigSet。

要使用此API创建的ConfigSet作为集合的配置，请使用Collections API。

该API只能与SolrCloud模式下运行的Solr一起使用。 如果您没有在SolrCloud模式下运行Solr，但仍希望使用共享配置，请参阅“配置集”一节。

### API Entry Points

The base URL for all API calls is http://<hostname>:<port>/solr.
* /admin/configs?action=CREATE: create a ConfigSet, based on an existing ConfigSet
* /admin/configs?action=DELETE: delete a ConfigSet
* /admin/configs?action=LIST: list all ConfigSets
* /admin/configs?action=UPLOAD: upload a ConfigSet

### Create a ConfigSet

`/admin/configs?action=CREATE&name=name&baseConfigSet=baseConfigSet`

根据现有的ConfigSet创建一个ConfigSet

参数

|Key|Type|Required|Default|Description|
|----|----|-----|-----|-----|
|name|string|Yes||被创建的ConfigSet|
|baseConfigSet|string|Yes|字面意思|
|configSetProp.name=value|string|No|覆盖基本baseConfigSet的属性键值对|

### Delete a ConfigSet

`/admin/configs?action=DELETE&name=name`

参数

|Key|Type|Required|Default|Description|
|----|----|-----|-----|-----|
|name|string|Yes||被删除的ConfigSet|

### List ConfigSets

`/admin/configs?action=LIST`，就不说了

### Upload a ConfigSet

`/admin/configs?action=UPLOAD&name=name`

上传一个ConfigSet，以压缩文件形式发送。请注意，如果启用了身份验证，则会以“受信任”模式上传ConfigSet，并将此上传操作作为经过身份验证的请求执行。
没有身份验证，ConfigSet将以“不受信任”模式上传。 使用“不受信任”ConfigSet创建集合后，以下功能将无法正常工作：

* 如果在ConfigSet中指定`RunExecutableListener`，但不会初始化。
* 如果在ConfigSet中指定`DataImportHandler’s ScriptTransformer`，但不会初始化。
* XSLT转换器（tr参数）在请求处理时间不能使用。
* 如果在ConfigSet中指定`StatelessScriptUpdateProcessor`，但不会初始化。

参数

|Key|Type|Required|Default|Description|
|----|----|-----|-----|-----|
|name|string|Yes||被创建的ConfigSet|
