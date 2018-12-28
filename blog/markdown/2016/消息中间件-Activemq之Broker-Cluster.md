接着上一篇[消息中间件-Activemq之Master-Slaver](http://my.oschina.net/OutOfMemory/blog/596343)，下面看看Broker-Cluster实现负载均衡  
Broker-Cluster实现负载均衡

Broker-Cluster部署方式中，各个broker通过网络互相连接，并共享queue，提供了2中部署方式：  
static Broker-Cluster和Dynamic Broker-Cluster

1).static Broker-Cluster

将ActiveMq拷贝2份，分别命名:apache-activemq-5.10.0\_M1,apache-activemq-5.10.0\_M2，  
下面就是配置activemq,xml:  
M1做如下配置:  

```
<networkConnectors>
	<networkConnector uri="static:(tcp://localhost:61617)"/>
</networkConnectors>
```

```
<transportConnectors>
        <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnectio        ns=1000&amp;wireFormat.maxFrameSize=104857600"/>
 </transportConnectors>
```

M2做如下配置:  

```
<networkConnectors>
	<networkConnector uri="static:(tcp://localhost:61616)"/>
</networkConnectors>
```

```
<transportConnectors>
        <transportConnector name="openwire" uri="tcp://0.0.0.0:61617?maximumConnectio        ns=1000&amp;wireFormat.maxFrameSize=104857600"/>
 </transportConnectors>
```

通过以上配置使M1和M2这两个 broker通过网络互相连接，并共享queue，  
启动M1和M2,可以看到如下启动日志：  
![](http://static.oschina.net/uploads/space/2016/0103/194536_ySvR_159239.png)  

![](http://static.oschina.net/uploads/space/2016/0103/194609_AlHc_159239.png)  

可以看到M1和M2，network connection has been established

测试我们还是用上一篇中的Sender和Receiver类，只需要做一点点修改：  
Sender类还是链接tcp://localhost:61616，发送消息到queue，  
Receiver做如下修改:  

```
connectionFactory = new ActiveMQConnectionFactory(
	ActiveMQConnection.DEFAULT_USER,
	ActiveMQConnection.DEFAULT_PASSWORD, "tcp://localhost:61617");
```

经测试Receiver可以接受到数据，表示M1和M2已经共享了queue

2).Dynamic Broker-Cluster

Dynamic Discovery集群方式在配置ActiveMQ实例时，不需要知道所有其它实例的URI地址  
对activemq.xml做如下配置:  
M1做如下配置:  

```
<networkConnectors>
	<networkConnector uri="multicast://default"/>
</networkConnectors>
```

```
<transportConnectors>
        <transportConnector name="openwire" uri="tcp://0.0.0.0:61616? maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"
          discoveryUri="multicast://default"/>
 </transportConnectors>
```

M2做如下配置:

```
<networkConnectors>
	<networkConnector uri="multicast://default"/>
</networkConnectors>
```

```
<transportConnectors>
        <transportConnector name="openwire" uri="tcp://0.0.0.0:61617? maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"
          discoveryUri="multicast://default"/>
 </transportConnectors>
```

启动M1和M2,可以看到如下启动日志： network connection has been established

测试同static broker-cluster,可以得到相同的结果。  
官网配置说明：[http://activemq.apache.org/networks-of-brokers.html](http://activemq.apache.org/networks-of-brokers.html)

Master-Slaver保证了数据的可靠性，Broker-Cluster提供了负载均衡，所以一般正式环境中都会采用：  
Master-Slaver+Broker-Cluster的模式