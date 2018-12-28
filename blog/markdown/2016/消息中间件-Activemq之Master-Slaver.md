什么叫中间件?  
中间件为软件应用提供了操作系统所提供的服务之外的服务，可以把中间件描述为"软件胶水"。中间件不是操作系统的一部分，不是数据库操作系统，也不是应用软件的一部分，而是能够让软件开发者方便的处理通信、输入和输出，能够专注自己应用的部分。

消息中间件解决了应用之间的消息传递、解耦、异步的问题。  
ActiveMQ 是Apache出品，最流行的，能力强劲的开源消息总线。ActiveMQ 是一个完全支持JMS1.1和J2EE 1.4规范的 JMS Provider实现，尽管JMS规范出台已经是很久的事情了，但是JMS在当今的J2EE应用中间仍然扮演着特殊的地位。

一般中间件都提供了横向扩展和纵向扩展，横向扩展就是我们经常说的负载均衡,纵向扩展提供了Master-Slaver;

负载均衡:提供负载均衡的中间件都对外提供服务  
Master-Slaver：同时只有一个中间件对外提供服务,当Master出现挂机等问题，Slaver会自动接管  

看一个整合的简图:  
![](http://static.oschina.net/uploads/space/2016/0102/163534_dbUd_159239.png)  

当然activemq也提供了以上2中方式，分别是:Master-Slave和Broker Cluster

准备：  
jdk1.6，apache-activemq-5.10.0，mysql5.1，zookeeper-3.4.3

先来看看Master-Slave模式

1).Shared File System Master Slave  
基于ActiveMQ的默认数据库kahaDB完成的，kahaDB的底层是文件系统。这种方式的集群，Slave的个数没有限制，哪个ActiveMQ实例先获取共享文件的锁，那个实例就是Master，其它的ActiveMQ实例就是Slave，当当前的Master失效，其它的Slave就会去竞争共享文件锁，谁竞争到了谁就是Master

本次测试在同一台机器上：  
首先更改配置conf/activemq,做如下修改：  

```
<persistenceAdapter>
        <!--<kahaDB directory="${activemq.data}/kahadb"/>-->
	<kahaDB directory="D:/kahaDB"/>
 </persistenceAdapter>
```

将activemq拷贝3份，分别：apache-activemq-5.10.0-M1， apache-activemq-5.10.0-M2，apache-activemq-5.10.0-M3，分别启动activemq命令，启动的日志分别是:  
![](http://static.oschina.net/uploads/space/2016/0102/165241_whka_159239.png)

表示当前进程是Master  
  
![](http://static.oschina.net/uploads/space/2016/0102/165334_GWQW_159239.png)  

表示当前进程没有获取到锁，作为Slaver  

测试：

下面的例子中分别提供了Producer(Sender类)和Consumer(Receiver类)  
我们首先用Producer发送消息给activemq，然后停止Master，然后再用Consumer接受消息，测试结果是可以接受到数据的。

2）.JDBC Master Slave  
JDBC Master Slave模式和Shared File Sysytem Master Slave模式的原理是一样的，只是把共享文件系统换成了共享数据库。

修改配置文件conf/activemq  

```
<persistenceAdapter>
        <!--<kahaDB directory="${activemq.data}/kahadb"/>-->
	<!--<kahaDB directory="D:/kahaDB"/>-->
        <jdbcPersistenceAdapter dataDirectory="${activemq.base}/data" dataSource="#mysql-ds"/> 
  </persistenceAdapter>
```

添加数据源：

```
<bean id="mysql-ds" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
	<property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
	<property name="username" value="root"/>
	<property name="password" value="root"/>
	<property name="poolPreparedStatements" value="true"/>
    </bean>
```

注：这里使用的是mysql，所以需要mysql驱动程序： mysql-connector-java-5.1.18，讲jar包放入lib文件夹下面，驱动版本不对，会出现如下错误: Database lock driver override not found for : \[mysql_connect ...

  

分别拷贝到其他几个文件夹下，分别启动，启动成功后我们可以看到数据库中多了几张表：

![](http://static.oschina.net/uploads/space/2016/0102/175717_XMFX_159239.png)  

测试方式同上；  
官网手册:[http://activemq.apache.org/jdbc-master-slave.html](http://activemq.apache.org/jdbc-master-slave.html)

3).Replicated LevelDB Store  
这种方式是ActiveMQ5.9以后才新增的特性，使用ZooKeeper协调选择一个node作为master

修改配置文件conf/activemq：  

```
<!--<persistenceAdapter>
         <kahaDB directory="${activemq.data}/kahadb"/>
         <kahaDB directory="D:/kahaDB"/>
         <jdbcPersistenceAdapter dataDirectory="${activemq.base}/data" dataSource="#mysql-ds"/> 
 </persistenceAdapter>-->
 <persistenceAdapter>
	<replicatedLevelDB
		directory="${activemq.data}/leveldb"
		replicas="3"
		bind="tcp://0.0.0.0:0"
		zkAddress="127.0.0.1:2181"
		hostname="127.0.0.1"
		sync="local_disk"
		zkPath="/activemq/leveldb-stores"/>
 </persistenceAdapter>
```

首先启动zookeeper，这里没有做集群处理,默认端口是2181，然后分别启动activemq，  
启动之后报错："activemq LevelDB IOException handler"。 

原因：版本5.10.0存在的依赖冲突。  

解决方案：  
（1）移除lib目录中的pax-url-aether-1.5.2.jar包  
（2）注释掉配置文件中的日志配置；

```
<!-- Allows accessing the server log
    <bean id="logQuery" class="org.fusesource.insight.log.log4j.Log4jLogQuery"
          lazy-init="false" scope="singleton"
          init-method="start" destroy-method="stop">
    </bean>
-->
```

测试方式同上；

提供java版的例子：  

```
public class Sender {
	private static final int SEND_NUMBER = 5;

	public static void main(String[] args) {
		ConnectionFactory connectionFactory;
		Connection connection = null;
		Session session;
		Destination destination;
		MessageProducer producer;
		connectionFactory = new ActiveMQConnectionFactory(
			ActiveMQConnection.DEFAULT_USER,
			ActiveMQConnection.DEFAULT_PASSWORD, "tcp://localhost:61616");
		try {
			connection = connectionFactory.createConnection();
			connection.start();
			session = connection.createSession(Boolean.TRUE,
					Session.AUTO_ACKNOWLEDGE);
			destination = session.createQueue("FirstQueue");
			producer = session.createProducer(destination);
			producer.setDeliveryMode(DeliveryMode.PERSISTENT);
			sendMessage(session, producer);
			session.commit();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				if (null != connection)
					connection.close();
			} catch (Throwable ignore) {
			}
		}
	}

	public static void sendMessage(Session session, MessageProducer producer)
			throws Exception {
		for (int i = 1; i <= SEND_NUMBER; i++) {
			TextMessage message = session.createTextMessage("发送的消息"
					+ i);
			System.out.println("发送消息：" + "ActiveMq 发送的消息" + i);
			producer.send(message);
		}
	}
}
```

```
public class Receiver {
	public static void main(String[] args) {
		ConnectionFactory connectionFactory;
		Connection connection = null;
		Session session;
		Destination destination;
		MessageConsumer consumer;
		connectionFactory = new ActiveMQConnectionFactory(
			ActiveMQConnection.DEFAULT_USER,
			ActiveMQConnection.DEFAULT_PASSWORD, "tcp://localhost:61616");
		try {
			connection = connectionFactory.createConnection();
			connection.start();
			session = connection.createSession(Boolean.FALSE,
					Session.AUTO_ACKNOWLEDGE);
			destination = session.createQueue("FirstQueue");
			consumer = session.createConsumer(destination);
			while (true) {
				TextMessage message = (TextMessage) consumer.receive(100000);
				if (null != message) {
					System.out.println("收到消息" + message.getText());
				} else {
					break;
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				if (null != connection)
					connection.close();
			} catch (Throwable ignore) {
			}
		}
	}
}
```