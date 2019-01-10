## RocketMQ����ṹ

![](https://oscimg.oschina.net/oscnet/b95324fc1ac6246060bdf2bb1e0edba92e3.jpg)

����ͼ��ʾ��������Էֳ�4����ɫ���ֱ��ǣ�Producer��Consumer��Broker�Լ�NameServer��

### 1.NameServer

�������Ϊ����Ϣ���е�Э���ߣ�Broker����ע��·����Ϣ��ͬʱClient�����ȡ·����Ϣ�����ʹ�ù�Zookeeper���ͱȽ���������ˣ����ǹ��ܱ�Zookeeper����  
NameServer������û��״̬�ģ����Ҷ��NameServerֱ�Ӳ�û��ͨ�ţ����Ժ�����չ��̨��Broker���ÿһ̨NameServer���������ӣ�

### 2.Broker

Broker��RocketMQ�ĺ��ģ��ṩ����Ϣ�Ľ��գ��洢����ȡ�ȹ��ܣ�һ�㶼��Ҫ��֤Broker�ĸ߿��ã����Ի�����Broker Slave����Master�ҵ�֮��ConsumerȻ���������Slave��  
Broker��ΪMaster��Slave��һ��Master���Զ�Ӧ���Slave��Master��Slave�Ķ�Ӧ��ϵͨ��ָ����ͬ��BrokerName����ͬ��BrokerId�����壬BrokerIdΪ0��ʾMaster����0��ʾSlave��

### 3.Producer

��Ϣ���е������ߣ���Ҫ��NameServer�������ӣ���NameServer��ȡTopic·����Ϣ�������ṩTopic�����Broker Master�������ӣ�Producer��״̬������Ⱥ����

### 4.Consumer

��Ϣ���е������ߣ�ͬ����NameServer�������ӣ���NameServer��ȡTopic·����Ϣ�������ṩTopic�����Broker Master��Slave�������ӣ�

### 5.Topic��Message Queue

�ڽ���������4����ɫ�Ժ󣬻���Ҫ�ص����һ�������ᵽ��Topic��Message Queue��������˼�������⣬�������ֲ�ͬ���͵���Ϣ�����ͺͽ�����Ϣǰ����Ҫ�ȴ���Topic�����Topic�����ͺͽ�����Ϣ��Ϊ��������ܺ���������������Message Queue��һ��Topic��������һ������Message Queue���е�����kafka�ķ���(Partition)��������Ϣ�Ϳ��Բ���������Message Queue������Ϣ��������Ҳ���Բ��еĴӶ��Message Queue��ȡ��Ϣ��

## �������úͲ���

���²�����**centos7��jdk1.8��rocketmq4.3.2**������RocketMQ��˳����������NameServer��Ȼ��������Broker��

### 1.NameServer����

```
[root@localhost bin]# ./mqnamesrv
Java HotSpot(TM) 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
Java HotSpot(TM) 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
The Name Server boot success. serializeType=JSON
```

������־��ʾ�����ɹ���Ĭ�϶˿�Ϊ9876��

### 2.Broker����

```
[root@localhost bin]# ./mqbroker
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000005c0000000, 8589934592, 0) failed; error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 8589934592 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /root/rocketmq-all-4.3.2-bin-release/bin/hs_err_pid3977.log
```

����ʧ�ܣ����ڴ治�㣬��Ҫ��rocketmqĬ�����õ���������ֵ�Ƚϴ��޸�runbroker.sh����

```
[root@localhost bin]# vi runbroker.sh

JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

Ĭ�����õĿ����ڴ�Ϊ8g��������ڴ治�����޸�Ϊ���¼���

```
JAVA_OPT="${JAVA_OPT} -server -Xms128m -Xmx128m -Xmn128m"
```

�ٴ���������־���£���ʾ�����ɹ���Ĭ�϶˿�Ϊ10911��

```
[root@localhost bin]# ./mqbroker
The broker[localhost.localdomain, 192.168.237.128:10911] boot success. serializeType=JSON
```

### 3.�򵥲���

#### 3.1������

```
public class SyncProducer {

?????public static void main(String[] args) throws Exception {
???????????// ����Producer
???????????DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
???????????producer.setNamesrvAddr("192.168.237.128:9876");
???????????// ��ʼ��Producer������Ӧ�����������ڣ�ֻ��Ҫ��ʼ��1��
???????????producer.start();
???????????for (int i = 0; i < 100; i++) {
????????????????Message msg = new Message("TopicTest", "TagA",
???????????????????????????("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
????????????????SendResult sendResult = producer.send(msg);
????????????????System.out.println(sendResult);
???????????}
???????????producer.shutdown();
?????}
}
```

������һ��DefaultMQProducer����ͬʱ������GroupName��NameServer��ַ��Ȼ�󴴽�Message��Ϣͨ��DefaultMQProducer����Ϣ���ͳ�ȥ������һ��SendResult����

#### 3.2������

```
public class PushConsumer {

?????public static void main(String[] args) throws MQClientException {
???????????DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please rename to unique group name");
?????????  consumer.setNamesrvAddr("192.168.237.128:9876");
???????????consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
???????????consumer.subscribe("TopicTest", "*");
???????????consumer.registerMessageListener(new MessageListenerConcurrently() {
????????????????public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
?????????????????????System.out.printf(Thread.currentThread().getName() + "Receive New Messages :" + msgs + "%n");
?????????????????????return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
????????????????}
???????????});
???????????consumer.start();
?????}
}
```

ͬ��ָ����GroupName��NameServer��ַ��������Topic��

#### 3.3���в���

ֱ�����������߱����´���

```
Exception in thread "main" org.apache.rocketmq.client.exception.MQClientException: No route info of this topic, TopicTest
See http://rocketmq.apache.org/docs/faq/ for further details.
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:634)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1253)
    at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1203)
    at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:214)
    at com.rocketmq.SyncProducer.main(SyncProducer.java:26)
```

������ʾ"û�д�Topic��·����Ϣ"��Ҳ�����������ڷ�����Ϣ��ʱ��û�л�ȡ��·����Ϣ���Ҳ���ָ����Broker,���ܵ�ԭ��  
1.Brokerû����ȷ����NameServer  
2.Producerû������NameServer  
3.Topicû�б���ȷ����  
SyncProducer��ָ����NameServer�ĵ�ַ��ͬʱRocketMQĬ������»��Զ�����Topic������ԭ����Brokerû��ע�ᵽNameServer������ָ��NameServer��������

```
[root@localhost bin]# ./mqbroker -n localhost:9876
The broker[localhost.localdomain, 192.168.237.128:10911] boot success. serializeType=JSON and name server is localhost:9876
```

�ٴ�����SyncProducer����־���£�

```
SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4C60000, offsetMsgId=C0A8ED8000002A9F000000000000229C, messageQueue=MessageQueue[topic=TopicTest, brokerName=localhost.localdomain, queueId=1], queueOffset=11]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4CD0001, offsetMsgId=C0A8ED8000002A9F000000000000234D, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=2], queueOffset=9]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4D90002, offsetMsgId=C0A8ED8000002A9F00000000000023FE, messageQueue=MessageQueue[topic=TopicTest, brokerName=localhost.localdomain, queueId=3], queueOffset=9]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4E80003, offsetMsgId=C0A8ED8000002A9F00000000000024AF, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=0], queueOffset=11]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4F40004, offsetMsgId=C0A8ED8000002A9F0000000000002560, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=1], queueOffset=12]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C4F70005, offsetMsgId=C0A8ED8000002A9F0000000000002611, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=2], queueOffset=10]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C5030006, offsetMsgId=C0A8ED8000002A9F00000000000026C2, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=3], queueOffset=10]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C5070007, offsetMsgId=C0A8ED8000002A9F0000000000002773, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=0], queueOffset=12]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C50A0008, offsetMsgId=C0A8ED8000002A9F0000000000002824, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=1], queueOffset=13]

SendResult [sendStatus=SEND_OK, msgId=0A0D53073B6073D16E933086C50D0009, offsetMsgId=C0A8ED8000002A9F00000000000028D5, messageQueue=MessageQueue [topic=TopicTest, brokerName=localhost.localdomain, queueId=2], queueOffset=11]
```

������ʹ�õ���pushģʽ������ʵʱ������Ϣ��

```
ConsumeMessageThread_13Receive New Messages :[MessageExt [queueId=1, storeSize=177,queueOffset=11, sysFlag=0, bornTimestamp=1547086138566, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430770, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F000000000000229C, commitLogOffset=8860, bodyCRC=705268097, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=12, CONSUME_START_TIME=1547086138573, UNIQ_KEY=0A0D53073B6073D16E933086C4C60000, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 48], transactionId='null'}]]

ConsumeMessageThread_3Receive New Messages :[MessageExt [queueId=2, storeSize=177, queueOffset=9, sysFlag=0, bornTimestamp=1547086138573, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430783, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F000000000000234D, commitLogOffset=9037, bodyCRC=1561245975, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1547086138598, UNIQ_KEY=0A0D53073B6073D16E933086C4CD0001, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 49], transactionId='null'}]]

ConsumeMessageThread_17Receive New Messages :[MessageExt [queueId=3, storeSize=177, queueOffset=9, sysFlag=0, bornTimestamp=1547086138585, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430794, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F00000000000023FE, commitLogOffset=9214, bodyCRC=1141369005, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1547086138601, UNIQ_KEY=0A0D53073B6073D16E933086C4D90002, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 50], transactionId='null'}]]

ConsumeMessageThread_9Receive New Messages :[MessageExt [queueId=0, storeSize=177, queueOffset=11, sysFlag=0, bornTimestamp=1547086138600, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430807, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F00000000000024AF, commitLogOffset=9391, bodyCRC=855693371, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=12, CONSUME_START_TIME=1547086138612, UNIQ_KEY=0A0D53073B6073D16E933086C4E80003, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 51], transactionId='null'}]]

ConsumeMessageThread_15Receive New Messages :[MessageExt [queueId=1, storeSize=177, queueOffset=12, sysFlag=0, bornTimestamp=1547086138612, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430809, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F0000000000002560, commitLogOffset=9568, bodyCRC=761548184, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=13, CONSUME_START_TIME=1547086138626, UNIQ_KEY=0A0D53073B6073D16E933086C4F40004, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 52], transactionId='null'}]]

ConsumeMessageThread_11Receive New Messages :[MessageExt [queueId=2, storeSize=177, queueOffset=10, sysFlag=0, bornTimestamp=1547086138615, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430820, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F0000000000002611, commitLogOffset=9745, bodyCRC=1516469518, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1547086138628, UNIQ_KEY=0A0D53073B6073D16E933086C4F70005, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 53], transactionId='null'}]]

ConsumeMessageThread_4Receive New Messages :[MessageExt [queueId=3, storeSize=177,queueOffset=10, sysFlag=0, bornTimestamp=1547086138627, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430824, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F00000000000026C2,commitLogOffset=9922, bodyCRC=1131031732,reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1547086138633, UNIQ_KEY=0A0D53073B6073D16E933086C5030006, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 54], transactionId='null'}]]

ConsumeMessageThread_14Receive New Messages :[MessageExt [queueId=0, storeSize=177, queueOffset=12, sysFlag=0, bornTimestamp=1547086138631, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430827, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F0000000000002773, commitLogOffset=10099, bodyCRC=879565858, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=13, CONSUME_START_TIME=1547086138635, UNIQ_KEY=0A0D53073B6073D16E933086C5070007, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 55], transactionId='null'}]]

ConsumeMessageThread_10Receive New Messages :[MessageExt [queueId=1, storeSize=177, queueOffset=13, sysFlag=0, bornTimestamp=1547086138634, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430830, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F0000000000002824, commitLogOffset=10276, bodyCRC=617742771, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=14, CONSUME_START_TIME=1547086138638, UNIQ_KEY=0A0D53073B6073D16E933086C50A0008, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 56], transactionId='null'}]]

ConsumeMessageThread_7Receive New Messages :[MessageExt [queueId=2, storeSize=177, queueOffset=11, sysFlag=0, bornTimestamp=1547086138637, bornHost=/192.168.237.1:53524, storeTimestamp=1547139430833, storeHost=/192.168.237.128:10911, msgId=C0A8ED8000002A9F00000000000028D5, commitLogOffset=10453, bodyCRC=1406480677, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=12, CONSUME_START_TIME=1547086138641, UNIQ_KEY=0A0D53073B6073D16E933086C50D0009, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 57], transactionId='null'}]]
```

## �����Ⱥ���úͲ���

�ֱ�����̨NameServer����̨Broker���ҷֱ��ṩSlave��׼����̨���Էֱ��Ǳ�����windows�Լ������centos��

### 1.����NameServer

�ֱ�����2̨NameServer���˿ںŶ�ʹ��Ĭ�ϵ�9876����ַ�˿����£�

```
192.168.237.128:9876
10.13.83.7:9876
```

### 2.����Broker

ÿ̨�����Ϸֱ�����һ��Master��һ��Slave����Ϊ����������Ŀ¼�µ�conf�ļ������ṩ�˶���broker����ģʽ���ֱ��У�2m-2s-async��2m-2s-sync��2m-noslave�������Դ�Ϊģ�����������ã�

#### 2.1����10.13.83.7Master��Slave

Master�������£�

```
namesrvAddr=192.168.237.128:9876;10.13.83.7:9876
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=SYNC_MASTER
flushDiskType=ASYNC_FLUSH
listenPort=10911
storePathRootDir=E:/rocketmq-all-4.3.2-bin-release/store-a-m
```

Slave�������£�

```
namesrvAddr=192.168.237.128:9876;10.13.83.7:9876
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
listenPort=10811
storePathRootDir=E:/rocketmq-all-4.3.2-bin-release/store-a-s
```

�ֱ�����������£�

```
E:\rocketmq-all-4.3.2-bin-release\bin>mqbroker -c E:\rocketmq-all-4.3.2-bin-rele
ase\conf\broker-m.conf
The broker[broker-a, 10.13.83.7:10911] boot success. serializeType=JSON and name
 server is 192.168.237.128:9876;10.13.83.7:9876
```

������Master������־��Slave��־���£�

```
E:\rocketmq-all-4.3.2-bin-release\bin>mqbroker -c E:\rocketmq-all-4.3.2-bin-rele
ase\conf\broker-s.conf
The broker[broker-a, 10.13.83.7:10811] boot success. serializeType=JSON and name
 server is 192.168.237.128:9876;10.13.83.7:9876
```

#### 2.2����10.13.83.7Slave

Master�������£�

```
namesrvAddr=192.168.237.128:9876;10.13.83.7:9876
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=SYNC_MASTER
flushDiskType=ASYNC_FLUSH
listenPort=10911
storePathRootDir=/root/rocketmq-all-4.3.2-bin-release/store-b-m
```

Slave�������£�

```
namesrvAddr=192.168.237.128:9876;10.13.83.7:9876
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
listenPort=10811
storePathRootDir=/root/rocketmq-all-4.3.2-bin-release/store-b-s
```

������־�ֱ����£�

```
[root@localhost bin]# ./mqbroker -c ../conf/broker-m.conf 
The broker[broker-b, 192.168.237.128:10911] boot success. serializeType=JSON and name server is 192.168.237.128:9876;10.13.83.7:9876
```

```
[root@localhost bin]# ./mqbroker -c ../conf/broker-s.conf 
The broker[broker-b, 192.168.237.128:10811] boot success. serializeType=JSON and name server is 192.168.237.128:9876;10.13.83.7:9876
```

### 3.����˵��

#### 1.namesrvAddr

NameServer��ַ���������ö�����ö��ŷָ���

#### 2.brokerClusterName

������Ⱥ���ƣ�����ڵ�϶�������ö��

#### 3.brokerName

broker���ƣ�master��slaveʹ����ͬ�����ƣ��������ǵ����ӹ�ϵ

#### 4.brokerId

0��ʾMaster������0��ʾ��ͬ��slave

#### 5.deleteWhen

��ʾ��������Ϣɾ��������Ĭ�����賿4��

#### 6.fileReservedTime

�ڴ����ϱ�����Ϣ��ʱ������λ��Сʱ

#### 7.brokerRole

������ֵ��SYNC\_MASTER��ASYNC\_MASTER��SLAVE��ͬ�����첽��ʾMaster��Slave֮��ͬ�����ݵĻ��ƣ�

#### 8.flushDiskType

ˢ�̲��ԣ�ȡֵΪ��ASYNC\_FLUSH��SYNC\_FLUSH��ʾͬ��ˢ�̺��첽ˢ�̣�SYNC\_FLUSH��Ϣд����̺�ŷ��سɹ�״̬��ASYNC\_FLUSH����Ҫ��

#### 9.listenPort

���������Ķ˿ں�

#### 10.storePathRootDir

�洢��Ϣ�ĸ�Ŀ¼

## ������

### 1.�����й�����

mqadmin��RocketMQ�Դ��������й����ߣ����Դ������޸�Topic����ѯ��Ϣ������������Ϣ�Ȳ������������ͨ����������鿴��

```
E:\rocketmq-all-4.3.2-bin-release\bin>mqadmin
The most commonly used mqadmin commands are:
   updateTopic          Update or create topic
   deleteTopic          Delete topic from broker and NameServer.
   updateSubGroup       Update or create subscription group
   deleteSubGroup       Delete subscription group from broker.
   updateBrokerConfig   Update broker's config
   updateTopicPerm      Update topic perm
   topicRoute           Examine topic route info
   topicStatus          Examine topic Status info
   topicClusterList     get cluster info for topic
   brokerStatus         Fetch broker runtime status data
   queryMsgById         Query Message by Id
   queryMsgByKey        Query Message by Key
   queryMsgByUniqueKey  Query Message by Unique key
   queryMsgByOffset     Query Message by offset
   printMsg             Print Message Detail
   printMsgByQueue      Print Message Detail
   sendMsgStatus        send msg to broker.
   brokerConsumeStats   Fetch broker consume stats data
   producerConnection   Query producer's socket connection and client vers
   consumerConnection   Query consumer's socket connection, client version
ubscription
   consumerProgress     Query consumers's progress, speed
   consumerStatus       Query consumer's internal data structure
   cloneGroupOffset     clone offset from other group.
   clusterList          List all of clusters
   topicList            Fetch all topic list from name server
   updateKvConfig       Create or update KV config.
   deleteKvConfig       Delete KV config.
   wipeWritePerm        Wipe write perm of broker in all name server
   resetOffsetByTime    Reset consumer offset by timestamp(without client
t).
   updateOrderConf      Create or update or delete order conf
   cleanExpiredCQ       Clean expired ConsumeQueue on broker.
   cleanUnusedTopic     Clean unused topic on broker.
   startMonitoring      Start Monitoring
   statsAll             Topic and Consumer tps stats
   allocateMQ           Allocate MQ
   checkMsgSendRT       check message send response time
   clusterRT            List All clusters Message Send RT
   getNamesrvConfig     Get configs of name server.
   updateNamesrvConfig  Update configs of name server.
   getBrokerConfig      Get broker config by cluster or special broker!
   queryCq              Query cq command.
   sendMessage          Send a message
   consumeMessage       Consume message

See 'mqadmin help <command>' for more information on a specific command.
```

�г�������֧�ֵ������Լ��򵥵Ľ��ܣ�����뿴��ϸ�Ŀ����������

```
E:\rocketmq-all-4.3.2-bin-release\bin>mqadmin help statsAll
usage: mqadmin statsAll [-a] [-h] [-n <arg>] [-t <arg>]
 -a,--activeTopic         print active topic only
 -h,--help                Print help
 -n,--namesrvAddr <arg>   Name server address list, eg: 192.168.0.1:9876;192.168
.0.2:9876
 -t,--topic <arg>         print select topic only
```

### 2.ͼ�ν��������

����������ṩ��ͼ�ν�������ߣ���RocketMQ����չ�����棬�����ַ���£�

```
https://github.com/apache/rocketmq-externals/tree/release-rocketmq-console-1.0.0/rocketmq-console
```

Ŀǰ���ȶ��汾��1.0.0���������������ڱ������У���application.properties�������ã�

```
rocketmq.config.namesrvAddr=10.13.83.7:9876
```

��Ҫָ��NameServer�ĵ�ַ��Ȼ��Ϳ��Դ�������ˣ�����֮�������8080�˿ڣ�ֱ�ӷ��ʵ�ַ��

```
http://localhost:8080
```

![](https://oscimg.oschina.net/oscnet/4e6af5773de8faa6f4216a80e3d5c9f0811.jpg)

## �ܽ�

���Ĵ���򵥵İ�װ�������֣����Գ��õ����ò������˼򵥽��ܣ�Ȼ���˽���RocketMQ�Ĳ��������ṹ���ֱ�����еĽ�ɫ���˼򵥽��ܣ�������������RocketMQ�Ĺ����ߣ������RocketMQ�ļ�غ͹���