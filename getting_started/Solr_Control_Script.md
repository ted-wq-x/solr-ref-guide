## start/restart可选的参数

* bin/solr start [options]
* bin/solr start  -help
* bin/solr restart [options]
* bin/solr restart -help

如果没有正在运行的节点，运行restart，则会跳过stop，执行start

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
| -a "string" |添加solr的jvm参数|bin/solr start -a "-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=1044"|
| -cloud|在solrCloud模式下启动，-c为简写，默认启动内置的zookeeper，使用-z指定自定义的zk|bin/solr start -c|           
| -d <dir> |定义服务的根目录，默认为$SOLR_HOME/server,不常用|bin/solr start -d newServerDir|
| -e <name> |启动solr自带的例子,cloud,techproducts,dih,schemaless|bin/solr start -e schemaless|
| -f|启动后台线程运行，但是不能和-e参数同时使用|bin/solr start -f|
| -h <hostname> |定义solr的域名，默认为localhost|bin/solr start -h search.mysolr.com|
| -m <memory> |定义jvm的堆栈大小，该值为min（-Xms）和max(-Xmx)的值|bin/solr start -m 1g|
| -noprompt|抑制警告和提示|bin/solr start -e cloud -noprompt|
| -p <port> |solr端口,默认为8893|bin/solr start -p 8655|
| -s <dir> |设置solr.solr.home系统属性;定义创建的core的目录路径|bin/solr start -s newHome|
| -v |将日志级别设置为debug|bin/solr start -f -v|
| -q |将日志级别设置为warn|bin/solr start -f -q|
| -v | 启动solr时输出详细详细|bin/solr start -V|
| -z <zkHost> |在使用-c的情况下，使用该参数定义zk的地址|bin/solr start -c -z server1:2181,server2:2181|
| -force |如果尝试以root用户身份启动Solr，脚本将退出并发出警告运行Solr作为“root”可以导致问题。此时使用|sudo bin/solr start -force|

## 设置java的系统参数

使用额外的参数-D，例如定义自动的软提交的频率为3s
`bin/solr start -Dsolr.autoSoftCommit.maxTime=3000`

## stop的可选参数

* bin/solr stop [options]
* bin/solr stop -help

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
| -p <port> |指定停止的端口|bin/solr stop -p 8983|
| -all | 停止所有的端口| bin/solr stop -all|
| -k <key> | 保护，因为疏忽造成的停止，默认为“solrrocks”|bin/solr stop -k solrrocks|

## 系统参数

### version

`bin/solr version`，不解释

### status

`bin/solr status`，不解释

### Healthcheck

* bin/solr healthcheck [options]
* bin/solr healthcheck -help

参数：

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
| -c <collection> |运行健康检查的core名称，必填|bin/solr healthcheck -c gettingstarted|
| -z <zkhost> |指定zk的地址，默认为localhost:9983|bin/solr healthcheck -z localhost:2181|
                
在cloud模式下，提供各个节点的状态信息

## Create Collections and Cores

* bin/solr create [options]
* bin/solr create -help
         
| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
| -c <name> |创建的集合名称，必须|bin/solr create -c mycollection|
| -d <confdir> |core的配置文件目录，默认为data_driven_schema_configs|bin/solr create -d basic_configs|         
| -n <configName> |配置的名称，默认何core或者collection名称相同|bin/solr create -n basic|
| -p <port> |命令发送到的端口，默认为solr实例的端口|bin/solr create -p 8983|    
| -s <shards> |在cloud模式下的集合分片数，默认为1|bin/solr create -s 2|
| -rf <replicas> |集合中文档的备份数量，默认为1（不备份）|bin/solr create -rf 2|
| -force |同上|----|
     

#### cloud模式的配置目录

在cloud模式下，创建core之前，必须先将配置文件上传到zookeeper中，你需要做出的主要决定是ZooKeeper配置目录是否在应该在多个集合之间共享。
  
我们通过几个例子来说明配置目录在SolrCloud中的工作原理。

首先，你不需要提供-n或-d选项，然后默认的配置（$SOLR_HOME/server/solr/configsets/data_driven_schema_configs/conf）被上传到zookeeper，
且集合的名称相同，例如，以下命令的结果是将`data_driven_schema_config`配置文件上传到zookeeper的`/configs/contacts`当中：`bin/solr create -c contacts`。
如果你创建第二个，还是需要重新上传，每个core之间互不影响。

你可以使用-n选项覆盖ZooKeeper中配置目录的名称。 对于实例，命令`bin /solr create -c logs -d basic_configs -n basic`将上传
`server/solr/configsets/basic_configs/conf`目录到ZooKeeper中的`/configs/basic`

注意，可以使用-d选项指定和默认不同的配置，solr提供一些默认的配置在`server/solr/configsets`目录当中，但是也可以提供自己的配置目录路径使用-d选项。
例如，命令`bin/solr create -c mycoll -d /tmp/myconfigs`,将会上传`/tmp/myconfigs`目录到zookeeper的`/configs/mycoll`目录当中。
重申，除非您使用-n选项覆盖它，否则配置目录以集合命名。

`data_driven_schema_configs`中的schema在当数据被索引时会发生改变，因此，我们建议您不会在集合之间共享data-driven的配置，除非您确定将数据索引到其中一个集合时，所有集合也应该到相应的更改。

## Delete Collections and Cores
  
使用该命令的前提是solr实例已经启动                                                
* bin/solr delete [options]
* bin/solr delete -help                                                     
                                                                 
在cloud模式下，delete会首先检查配置目录是否有被其他collection使用，若没有，则会删除zookeeper上相应的配置。

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |
| -c <name> |要删除的集合名称,必须|bin/solr delete -c mycoll|
| -deleteConfig <true/false> |是否删除zookeeper上的配置文件，默认为true,若配置文件被别的集合使用，即使为true也不会删除|bin/solr delete -deleteConfig false|
| -p <port> |同上|---|
 
## Authentication

命令行的身份认证，目前只能在cloud模式下使用。

基础的认证`bin/solr auth enable/disable`,当使用用户界面或使用`bin/solr`或其他任何api请求时。

提示：具体的还是查看官方文档的`Securing Solr`章节。

1. 上传一个security.json文件到zookeeper
```json
{
  "authentication": {
    "blockUnknown": false,
    "class": "solr.BasicAuthPlugin",
    "credentials": {
      "user": "vgGVo69YJeUg/O6AcFiowWsdyOUdqfQvOLsrpIPMCzk=7iTnaKOWe+Uj5ZfGoKKK2G6hrcF10h6xezMQK+LBvpI="
    }
  },
  "authorization": {
    "class": "solr.RuleBasedAuthorizationPlugin",
    "permissions": [
      {
        "name": "security-edit",
        "role": "admin"
      },
      {
        "name": "collection-admin-edit",
        "role": "admin"
      },
      {
        "name": "core-admin-edit",
        "role": "admin"
      }
    ],
    "user-role": {
      "user": "admin"
    }
  }
}
```
                                                                                             
2. 添加两行代码到`bin/solr.in.sh` 或者 `bin\solr.in.cmd`,设置认证的类型和`basicAuth.conf`文件的路径 
```
# The following lines added by ./solr for enabling BasicAuth
SOLR_AUTH_TYPE="basic"
SOLR_AUTHENTICATION_OPTS="-Dsolr.httpclient.config=/path/to/solr-
6.6.0/server/solr/basicAuth.conf"
```        

3. 创建`server/solr/basicAuth.conf`文件，该文件存储在使用bin/solr时需要的凭证信息。

命令使用如下参数：
* -credentials
    > username:password
    > 如果你不想使用用户名和密码作为脚本参数的形式，可以使用-prompt参数。-prompt和-credentials必须有一个。

* -prompt
    > 若为true，将会提示你输入用户名和密码。-prompt和-credentials必须有一个。  

* -blockUnknown
    >　如果为true，则阻止所有未经身份验证的用户访问Solr。默认为false，这意味着未经身份验证的用户仍然可以访问Solr．

* -updateIncludeFileOnly     
    > 如果为true,只有bin /solr.in.sh或bin \solr.in.cmd中的设置将被更新，而security.json将不会被创建
             
* -z 
    > zookeeper的地址

* -d
    > 定义solr实例的目录,默认为$SOLR_HOME/server
   
* -s 
    > 定义solr.solr.home的位置，默认情况下是server /solr。 如果有多个实例的Solr在同一台主机上，或者如果您已经定制了
    $SOLR_HOME目录路径，则可能需要定义这个。

4. 取消基础认证     
     
bin/solr auth disable

如果-updateIncludeFileOnly选项为true，那只有bin/solr.in.sh或者bin\solr.in.cmd配置会被更新，security.json不会删除。

如果-updateIncludeFileOnly选项为false，bin/solr.in.sh或者bin\solr.in.cmd配置会被更新，且security.json会被删除，但是basicAuth.conf文件都不会被删除。


## zookeeper operations

solr脚本允许运行zk的相关的子命令                                   

* bin/solr zk [sub-command] [options]
* bin/solr zk -help

在使用这些命令之前，必须已经初始化solr和zookeeper实例，zookeeper一旦初始化之后，solr就不需要运行了，但任然能够运行这些命令。

### 上传配置

使用`zk upconfig` 命令上传配置到zookeeper。

| 参数 | 描述 | 例子 |
| ---- | :---- | ----- |  
| -n <name> |在zk中的configs节点下配置集合（可以理解为文件夹，但不是）的名称|-n myconfig|
| -d <configset dir> |待上传配置文件的位置，如果只提供文件夹名称，则默认在$SOLR_HOME/server/solr/configsets下查找|-d /path/to/configset/source|      
| -z <zkHost> | zk的地址，如果在脚本文件中配置了，可以忽略|-z 123.321.23.43:2181|

举个 :chestnut:

`bin/solr zk upconfig -z 111.222.333.444:2181 -n mynewconfig -d /path/to/configset`
          
                                      

                            