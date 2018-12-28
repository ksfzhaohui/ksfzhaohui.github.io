**安装Hadoop和Hbase**

hadoop和hbase版本的选择以及安装，参考之前的文章：

Hadoop的版本选择和单机模式：[http://codingo.xyz/index.php/2016/08/16/hadoop-stand-alone/](http://codingo.xyz/index.php/2016/08/16/hadoop-stand-alone/)

Hadoop的伪分布式模式：[http://codingo.xyz/index.php/2016/08/16/hadoop\_false\_distribute/](http://codingo.xyz/index.php/2016/08/16/hadoop_false_distribute/)

Hbase版本选择和单机模式入门：[http://codingo.xyz/index.php/2016/08/17/hbase_standalone/](http://codingo.xyz/index.php/2016/08/17/hbase_standalone/)

Hbase的伪分布式模式：[http://codingo.xyz/index.php/2016/08/16/hadoop\_false\_distribute/](http://codingo.xyz/index.php/2016/08/16/hadoop_false_distribute/)

Hadoop版本：**hadoop-2.5.1**

Hbase版本：**Hbase1.2.2**

**连接Hbase**

1.导入jar包

将HBASE_HOME/lib下的jar包加入classpath目录下

2.添加日志文件

为了能详细的了解输出日志，添加log4j.properties文件

3.添加hbase-site.xml

将HBASE_HOME/conf目录下hbase-site.xml文件拷贝到classpath目录下

```
<configuration>
	<property>
		<name>hbase.rootdir</name>
		<value>hdfs://192.168.111.129:9000/hbase</value>
	</property>
	<property>
		<name>hbase.cluster.distributed</name>
		<value>true</value>
	</property>
	<property>
		<name>hbase.zookeeper.quorum</name>
		<value>192.168.111.129</value>
	</property>
</configuration>
```

192.168.111.129 是本地虚拟机的ip，根据实际情况修改

**体系结构**

![](https://static.oschina.net/uploads/space/2016/0914/163455_vx41_159239.jpg)

来自Hbase权威指南的一张图片

**Hbase Client连接Server流程：**

首先HBase Client端会连接Zookeeper Qurom，通过Zookeeper组件Client能获知哪个Server管理-ROOT- Region。那么Client就去访问管理-ROOT-的Server，在META中记录了HBase中所有表信息，(你可以使用 scan 'hbase:meta' 命令列出你创建的所有表的详细信息)，从而获取Region分布的信息。一旦Client获取了这一行的位置信息，比如这一行属于哪个Region，Client将会缓存这个信息并直接访问HRegionServer。久而久之Client缓存的信息渐渐增多，即使不访问hbase:meta表也能知道去访问哪个HRegionServer。

所以需要做如下配置：

1.配置hbase.zookeeper.quorum

```
<property>
	<name>hbase.zookeeper.quorum</name>
	<value>192.168.111.129</value>
</property>
```

因为首先连接的就是Zookeeper Qurom，所以需要配置ip地址，同时需要对外打开2181端口，否则出现如下错误：

```
2016-09-14 16:27:21 - Opening socket connection to server 127.0.0.1/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
2016-09-14 16:27:22 - Session 0x0 for server null, unexpected error, closing socket connection and attempting reconnect
java.net.ConnectException: Connection refused: no further information
	at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
	at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:739)
	at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:361)
	at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1081)
```

2.配置hostname

配置完Zookeeper Qurom，执行client程序会有如下问题：

```
Call exception, tries=11, retries=35, started=60140 ms ago, cancelled=true, msg=row 'table1,row1,99999999999999' on table 'hbase:meta' at region=hbase:meta,,1.1588230740, hostname=bogon,16201,1473841196735, seqNum=0
```

C:\\Windows\\System32\\drivers\\etc的hosts文件加上一句：

```
192.168.111.129  bogon

```

bogon是服务器hostname

3.开启RegionServer端口

因为最后Hbase Client需要和指定的RegionServer进行连接，所以需要开启RegionServer端口，默认端口是：16201，否则有如下错误：

```
2016-09-14 11:26:33 [main] 405::: ERROR org.apache.hadoop.hbase.client.AsyncProcess - Failed to get region location 
org.apache.hadoop.hbase.TableNotFoundException: testtable
	at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.locateRegionInMeta(ConnectionManager.java:1283)
	at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.locateRegion(ConnectionManager.java:1181)
	at org.apache.hadoop.hbase.client.AsyncProcess.submit(AsyncProcess.java:395)
	at org.apache.hadoop.hbase.client.AsyncProcess.submit(AsyncProcess.java:344)
	at org.apache.hadoop.hbase.client.BufferedMutatorImpl.backgroundFlushCommits(BufferedMutatorImpl.java:238)
	at org.apache.hadoop.hbase.client.BufferedMutatorImpl.flush(BufferedMutatorImpl.java:190)
	at org.apache.hadoop.hbase.client.HTable.flushCommits(HTable.java:1434)
	at org.apache.hadoop.hbase.client.HTable.put(HTable.java:1018)
	at ch03.PutExample1.main(PutExample1.java:37)
```

测试实例：

```
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.util.Bytes;

import common.HBaseHelper;

public class PutExample {

	public static void main(String[] args) throws IOException {
		Configuration conf = HBaseConfiguration.create();
		HBaseHelper helper = HBaseHelper.getHelper(conf);
		Connection connection = ConnectionFactory.createConnection(conf);
		Table table = connection.getTable(TableName.valueOf("testtable"));
		Put put = new Put(Bytes.toBytes("row1"));

		put.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual3"),
				Bytes.toBytes("val1"));
		put.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual2"),
				Bytes.toBytes("val2"));

		table.put(put);
		table.close();
		connection.close();
		helper.close();
	}
}
```

注：以上的‘testtable’是在hbase shell中创建好的

测试结果：

```
hbase(main):006:0> scan 'testtable'
ROW                                        COLUMN+CELL                                                                                                                 
 row1                                      column=colfam1:qual2, timestamp=1473847426654, value=val2                                                                   
 row1                                      column=colfam1:qual3, timestamp=1473847426654, value=val1                                                                   
1 row(s) in 0.1690 seconds
```