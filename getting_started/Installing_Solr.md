由于Java的跨平台性，所以solr的安装，在所有的平台基本都能适用，但在windows上小概率的出现问题。

## 安装java

需要的JRE版本为1.8以上，不会安装的自信google。

## 安装solr

从[官网](http://lucene.apache.org/solr/)下载,解压即可

## 运行solr

安装目录下运行,对于Linux `bin/solr start`，对于windows `bin\solr.cmd start`。默认监听端口8983。

## solr的脚本参数(以linux为例)

* bin/solr -help,不解释
* bin/solr start -help，也不解释
* bin/solr start -f ,后台进程运行
* bin/solr start -p 8888 ,运行solr，指定监听端口
* bin/solr stop -p 8888 ,停止8888端口的solr
* bin/solr -e techproducts ,运行自带的例子
* bin/solr status ,查询solr的运行状态

## 创建core

bin/solr create -c <coreName>

## 添加文档

文档是solr查询的基础，类似于关系型数据库的一条记录。

solr的例子都在安装目录的example/子目录下，在/bin目录下有一个post脚本，该脚本能将各种内容post到solr当中，包括solr原生的xml和json格式，csv文件，丰富文档的目录树，甚至是简短的网络爬网
例如：`bin/post -c gettingstarted example/exampledocs/*.xml`

## 查询的方法，参考小节[searching](../searching/README.md)