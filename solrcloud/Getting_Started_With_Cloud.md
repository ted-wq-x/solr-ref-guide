SolrCloud旨在提供高可用，容错环境，用于分发您的索引跨多个服务器的内容和查询请求。

关于cloud的介绍还是看文档吧！我就不翻译了（毕竟太懒 :tongue: ）。

## SolrCloud example

1. 启动`bin/solr -e cloud`,然后出现一串的提示，自己看吧！

2. 文档中的相关命令在[Solr_Control_Script](../getting_started/Solr_Control_Script.md)中都有介绍，这里就不翻译了，自己去看文档吧！

## Adding a node to a cluster

添加一个新的机器节点到集群当中

`bin/solr start -cloud -s solr.home/solr -p <port num> -z <zk hosts string>`

注意：上面的命令需要一个新的Solr home，您需要将solr.xml复制到solr_home目录，或者保持在ZooKeeper /solr.xml中。


                        