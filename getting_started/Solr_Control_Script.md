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






         
          
         
          
                                      

                            