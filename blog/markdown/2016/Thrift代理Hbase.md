**使用HBase的2种方式：**

1.直接使用HBase客户端API，这样就限制了只能使用java语言

2.使用一些能够将请求转换成API的代理，这些代理将原始Java API包装成其他协议，这样客户端可以使用API提供的任意外部语言来编写程序。外部API实现了专门基于java的服务，而这种服务能够在内部使用由HTable客户端提供的API。

HBase本身对代理模式的支持也很广泛，比如支持的类型有：REST、Thrift、Avro等

关于代理的模式，可以看一张网上的架构图：

![](https://static.oschina.net/uploads/space/2016/0922/165757_rk65_159239.png)

这里主要介绍一下Thrift作为HBase的代理对外提供服务，主要是Thrift在性能上的优势以及对各种主流语言的支持

**1.安装HBase和Hadoop**

Hadoop版本：2.5.1  
Hbase版本：1.2.2

参考之前的文章：  
Hadoop的版本选择和单机模式：[http://codingo.xyz/index.php/2016/08/16/hadoop-stand-alone/](http://codingo.xyz/index.php/2016/08/16/hadoop-stand-alone/)  
Hadoop的伪分布式模式：[http://codingo.xyz/index.php/2016/08/16/hadoop\_false\_distribute/](http://codingo.xyz/index.php/2016/08/16/hadoop_false_distribute/)  
Hbase版本选择和单机模式入门：[http://codingo.xyz/index.php/2016/08/17/hbase_standalone/](http://codingo.xyz/index.php/2016/08/17/hbase_standalone/)  
Hbase的伪分布式模式：[http://codingo.xyz/index.php/2016/08/16/hadoop\_false\_distribute/](http://codingo.xyz/index.php/2016/08/16/hadoop_false_distribute/)

**2.安装Thrift**  
Thrift版本：0.9.3  
下载地址：[https://thrift.apache.org/download](https://thrift.apache.org/download)  
windows平台：直接使用thrift-0.9.3.exe  
其他平台安装：[https://thrift.apache.org/docs/install/](https://thrift.apache.org/docs/install/)

**3.编译模式文件**  
Hbase提供了Thrift需要的模式文件，存放在Hbase的源码中，需要下载的是：**hbase-1.2.2-src.tar.gz**  
路径：  
$HBASE_HOOME/src/main/resources/org/apache/hadoop/hbase/thrift/Hbase.thrift 和  
$HBASE_HOOME/src/main/resources/org/apache/hadoop/hbase/thrift2/hbase.thrift  
提供了2套Thrift文件，它们并不兼容；根据官方文档，thrift1很可能被抛弃，所有下面的例子中使用thrift2

windows平台编码模式文件：

```
thrift-0.9.3.exe --gen java hbase.thrift
```

其实java可以根据不同语言进行选择，比如c++、perl、php、python、ruby等  
生成了如下结构代码

![](https://static.oschina.net/uploads/space/2016/0922/170243_WMr7_159239.jpg)

拷贝到eclipse的开发环境中，这里没有直接使用官方提供的**hbase-thrift-1.2.2.jar**，jar文件中其实就是我们上面的类文件。

**4.获取Thrift提供的语言支持库**  
以上自动生成的类文件依赖于Thrift提供的支持库，thrift-0.9.3/lib目录下提供了各种语言的支持库  
选择java支持库，将其他的代码同样拷贝到eclipse的开发环境中，这里也没有使用官方提供的：**libthrift-0.9.3.jar**

**5.启动服务**  
以上提供了Hbase和Hadoop的伪分布式模式的安装和启动  
Hbase提供了对ThriftServer启动的支持

非守护模式启动和停止Thrift服务：

```
hbase thrift2 start
hbase thrift2 stop
```

后台进程启动和停止Thrift服务：

```
hbase-daemon.sh start thrift2
hbase-daemon.sh stop thrift2
```

启动日志：

```
[root@bogon ~]# hbase thrift2 start
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/root/hbase-1.2.2/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/root/hadoop-2.5.1/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
2016-09-20 08:57:05,211 INFO  [main] impl.MetricsConfig: loaded properties from hadoop-metrics2-hbase.properties
2016-09-20 08:57:05,367 INFO  [main] impl.MetricsSystemImpl: Scheduled snapshot period at 10 second(s).
2016-09-20 08:57:05,367 INFO  [main] impl.MetricsSystemImpl: HBase metrics system started
2016-09-20 08:57:06,123 INFO  [main] mortbay.log: Logging to org.slf4j.impl.Log4jLoggerAdapter(org.mortbay.log) via org.mortbay.log.Slf4jLog
2016-09-20 08:57:06,129 INFO  [main] http.HttpRequestLog: Http request log for http.requests.thrift is not defined
2016-09-20 08:57:06,148 INFO  [main] http.HttpServer: Added global filter 'safety' (class=org.apache.hadoop.hbase.http.HttpServer$QuotingInputFilter)
2016-09-20 08:57:06,148 INFO  [main] http.HttpServer: Added global filter 'clickjackingprevention' (class=org.apache.hadoop.hbase.http.ClickjackingPreventionFilter)
2016-09-20 08:57:06,151 INFO  [main] http.HttpServer: Added filter static_user_filter (class=org.apache.hadoop.hbase.http.lib.StaticUserWebFilter$StaticUserFilter) to context thrift
2016-09-20 08:57:06,151 INFO  [main] http.HttpServer: Added filter static_user_filter (class=org.apache.hadoop.hbase.http.lib.StaticUserWebFilter$StaticUserFilter) to context static
2016-09-20 08:57:06,152 INFO  [main] http.HttpServer: Added filter static_user_filter (class=org.apache.hadoop.hbase.http.lib.StaticUserWebFilter$StaticUserFilter) to context logs
2016-09-20 08:57:06,173 INFO  [main] http.HttpServer: Jetty bound to port 9095
2016-09-20 08:57:06,174 INFO  [main] mortbay.log: jetty-6.1.26
2016-09-20 08:57:06,680 INFO  [main] mortbay.log: Started SelectChannelConnector@0.0.0.0:9095
2016-09-20 08:57:06,695 INFO  [main] thrift2.ThriftServer: starting HBase ThreadPool Thrift server on 0.0.0.0/0.0.0.0:9090

```

默认对外提供的端口：**9090**

**6.测试**  
关于eclipse远程连接Hbase：[https://my.oschina.net/OutOfMemory/blog/746860](https://my.oschina.net/OutOfMemory/blog/746860)  
开发环境中已经有了编译模式文件生成的类文件以及Thrift提供的语言支持库

```
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.List;
import org.apache.hadoop.hbase.thrift2.generated.TColumnValue;
import org.apache.hadoop.hbase.thrift2.generated.THBaseService;
import org.apache.hadoop.hbase.thrift2.generated.TPut;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;

public class ThriftExample {

    public static void main(String[] args) throws Exception {
        TTransport transport = new TSocket("192.168.111.129", 9090, 20000);
        TProtocol protocol = new TBinaryProtocol(transport, true, true);
        THBaseService.Client client = new THBaseService.Client(protocol);
        transport.open();

        ByteBuffer table = ByteBuffer.wrap("testTable".getBytes());

        TPut put = new TPut();
        put.setRow("row1".getBytes());

        TColumnValue columnValue = new TColumnValue();
        columnValue.setFamily("family1".getBytes());
        columnValue.setQualifier("qualifier1".getBytes());
        columnValue.setValue("value1".getBytes());
        List<TColumnValue> columnValues = new ArrayList<TColumnValue>();
        columnValues.add(columnValue);
        put.setColumnValues(columnValues);

        client.put(table, put);
        transport.close();
    }
}
```

通过hbase shell查看：

```
hbase(main):003:0> scan 'testtable'
ROW                         COLUMN+CELL                                                
 rthrift                     column=colfam1:qualifier1, timestamp=1474376284145, value=value1
```

以上是用java语言进行的测试，以上的步骤只要在：**编译模式文件和Thrift提供的语言支持库** 进行稍微的修改也适用于其他语言

**个人博客：[codingo.xyz](http://codingo.xyz/)**