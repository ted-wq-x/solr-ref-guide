## solr运行时的根目录结构

Standalone Mode
```
<solr-home-directory>/
    solr.xml
    core_name1/
        core.properties
        conf/
            solrconfig.xml
            managed-schema
        data/
    core_name2/
        core.properties
        conf/
            solrconfig.xml
            managed-schema
        data/
```

SolrCloud Model
```
<solr-home-directory>/
    solr.xml
    core_name1/
        core.properties
        data/
    core_name2/
        core.properties
        data/
```

相关的文件描述：

* solr.xml为solr实例的配置文件
* 对于每个core：
    * core.properties中指定core的名称
    * solrconfig.xml控制更高级别的行为
    * managed-schema(或者schema.xml)定义core的索引，也是主要配置的地方
    * data/为索引文件目录
    
SolrCloud不包含conf/目录下的配置文件，这些文件存储在zookeeper当中。