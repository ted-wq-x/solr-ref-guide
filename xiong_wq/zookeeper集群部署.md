# zookeeper集群部署

## 伪集群配置

1. 在conf目录下新建zoo.cfg文件，文件类容如下

    ```
    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=D:\\Java\\soft\\zookeeper-3.4.6\\data\\1
    dataLogDir=D:\\Java\\soft\\zookeeper-3.4.6\\log\\1
    clientPort=2181
    server.1=127.0.0.1:2887:3887
    server.2=127.0.0.1:2888:3888
    server.3=127.0.0.1:2889:3889
    ```

    注意：参数不能少，如果少了会报错，注意看下启动时的日志。

    参数说明：
    * dataDir和dataLogDir顾名思义，都是文件夹.
    * tickTime:基本事件单元，以毫秒为单位。这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
    * clientPort:这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
    * syncLimit:发送请求和获取确认之间(心跳)确认连接之间的最大间隔数
    * initLimit:初始同步阶段可以使用的间隔数

    特殊属性：
    server.num=ip/domain：Port1：Port2
其中num：表示数字表示第几号服务器；ip/domain ：是服务器域名或者ip地址。Port1：表示这个服务器和集群中的Leader服务器交换信息的端口；Port2：表示万一集群中的Leader服务器挂了，需要一个端口重新进行选举，选出一个新的Leader，这个端口就是用来执行选举时服务器相互通信的端口。

2. 在dataDir下新建文件myid，内容为`server.1=127.0.0.1:2887:3887` 中server.x的x值。

3. 集群的每个zookeeper都需要配置上面的两个，注意修改相应的参数

4. 进入bin目录，在windows下，修改zkServer.cmd ，添加以下代码
`set ZOOCFG=E:\\work\\zk_03\\zookeeper-3.3.6\\conf\\zoo.cfg`，注意修改路径

## 集群的配置
 
只需要修改配置文件的端口就可以了。

## 基本命令

在Windows中使用zkServer.cmd直接启动，使用zkCli.cmd进行命令行操作

1. 显示根目录下、文件： ls / 使用 ls 命令来查看当前 ZooKeeper 中所包含的内容
2. 显示根目录下、文件： ls2 / 查看当前节点数据并能看到更新次数等数据
3. 创建文件，并设置初始内容： create /zk “test” 创建一个新的 znode节点“ zk ”以及与它关联的字符串
4. 获取文件内容： get /zk 确认 znode 是否包含我们所创建的字符串
5. 修改文件内容： set /zk “zkbak” 对 zk 所关联的字符串进行设置
6. 删除文件： delete /zk 将刚才创建的 znode 删除
7. 删除包含子文件夹：rmr /zk 将创建的zk目录已经它下面的子目录都删除
8. 退出客户端： quit
9. 帮助命令： help

使用set命令设置节点的数据，使用get获取与节点的数据，在不同的集群节点上可以看到数据在同步