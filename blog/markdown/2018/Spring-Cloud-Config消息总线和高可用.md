## **ϵ������**

[Spring-Cloud-Config���ٿ�ʼ](https://my.oschina.net/OutOfMemory/blog/1848235)

[Spring-Cloud-Config��Ϣ���ߺ͸߿���](https://my.oschina.net/OutOfMemory/blog/1865231)

## **ǰ��**

�����м򵥵Ľ�����Spring-Cloud-Config���ʹ�ã�����ֶ����������ļ�����������ĩ����˼������ʣ����а������Client�ڵ���θ��£�Server����α�֤�߿����Եȣ����Ľ��ص����ͨ��ʹ��Spring Cloud Bus���������¿ͻ��ˣ��Լ�Server��α�֤�߿��ã�

## **Spring Cloud Bus��Ϣ����**

Spring Cloud Busʹ����������Ϣ�������ӷֲ�ʽϵͳ�Ľڵ㣬�������ڹ㲥״̬�ı䣨���磬���øı䣩����������ָ�ĿǰΨһʵ�ֵķ�ʽ����AMQP��Ϣ������Ϊͨ������ʵ������������MQ�Ĺ㲥�����ڷֲ�ʽ��ϵͳ�д�����Ϣ��Ŀǰ���õ���Kafka��RabbitMQ�������ص�ʹ��kafka��ʵ�ֶ�ͻ���ˢ�������ļ���

### 1.�����������

��������ͼ������ʾ��  
![](https://oscimg.oschina.net/oscnet/71ff25fd61bab9d129cdd1d495a36fa932f.jpg)

### 2.kafka��װ����

kafka��������Zookeeper��ʹ�õİ汾�ֱ��ǣ�kafka_2.11-1.0.1��zookeeper-3.4.3��������ΰ�װ����ɲο���[Kafka���ٿ�ʼ](https://my.oschina.net/OutOfMemory/blog/827863)

### 3.server�˸���

3.1����µ�����

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

3.2application.properties�������

```
#������Ϣ����
spring.cloud.bus.trace.enabled=true
spring.cloud.stream.kafka.binder.brokers=192.168.237.128
spring.cloud.stream.kafka.binder.defaultBrokerPort=9092
```

### 4.client����

4.1����µ�����

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

4.2application.properties�������

```
#������Ϣ����
spring.cloud.bus.trace.enabled=true
spring.cloud.stream.kafka.binder.brokers=192.168.237.128
spring.cloud.stream.kafka.binder.defaultBrokerPort=9092
```

### 5.��������

5.1����Server��  
�۲�������־�����Է���/actuator/bus-refreshӳ��

```
2018-07-18 10:51:44.434 INFO 12532 --- [ main] s.b.a.e.w.s.WebMvcEndpointHandlerMapping : Mapped "{[/actuator/bus-refresh],methods=[POST]}" onto public java.lang.Objectorg.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping$OperationHandler.handle(javax.servlet.http.HttpServletRequest,java.util.Map<java.lang.String, java.lang.String>)
```

���¿���������������־��

```
2018-07-18 10:52:08.803 INFO 6308 --- [ main] o.s.c.s.b.k.p.KafkaTopicProvisioner : Using kafka topicfor outbound: springCloudBus
```

Server������kafka������һ������ΪspringCloudBus��Topic��������Ϊ�����ļ����µ���Ϣ֪ͨ������ȥkafka�ϲ鿴��

```
[root@localhost bin]# ./kafka-topics.sh --list --zookeeper 10.13.83.7:2181
__consumer_offsets
springCloudBus
```

5.2����Client  
�ֱ�ָ�������˿�Ϊ8881��8882�����Կ�����Server�����Ƶ���־����������ΪspringCloudBus��Topic������Server�˷�����Ϣ��kafka��kafka֪ͨclient�������ݣ�

5.3����  
�ֱ����[http://localhost](http://localhost/):8881/hello��[http://localhost](http://localhost/):8882/hello��������£�

```
hello test
```

����git�е������ļ�Ϊ��

```
foo=hello test update
```

POST��ʽ����Server�ˣ��������������ļ�

```
c:\curl-7.61.0\I386>curl -X POST http://localhost:8888/actuator/bus-refresh
```

�ֱ����[http://localhost](http://localhost/):8881/hello��[http://localhost](http://localhost/):8882/hello��������£�

```
hello test update
```

2���ͻ��˶���ȡ�������µ����ݣ���ʾ���³ɹ���

����ͼ�����Ƿ���Server�˳е���̫������񣬶���ͼ��Server����һ�����㣬�����Ͳ��ܱ�֤ϵͳ�߿��ã����濴һ����ηֲ�ʽ����Server�ˣ�

## **Server�˱�֤�߿���**

Server��ͨ��ע������Eureka����֤�߿��ã����濴һ�¾������̣�  
![](https://oscimg.oschina.net/oscnet/1cc9ad60fe256bc86e5596356be2e5b4a08.jpg)

### 1.Eurekaע������

1.1Eureka-Server����

```
<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
        <version>1.4.5.RELEASE</version>
</dependency>
```

1.2���������ļ�

```
spring.application.name=eureka-server
server.port=8880

eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8880/eureka/
```

eureka.client.register-with-eureka���Ƿ��Լ�ע�ᵽEureka Server��Ĭ��Ϊtrue  
eureka.client.fetch-registry���Ƿ��Eureka Server��ȡע����Ϣ��Ĭ��Ϊtrue  
eureka.client.serviceUrl.defaultZone��Eureka Server������ַ

1.3׼��������

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class, args);
    }
}
```

### 2.����Server�ˣ������ṩ����

2.1Eureka-Client����

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

2.2���������ļ�

```
eureka.client.serviceUrl.defaultZone=http://localhost:8880/eureka/
```

ָ��ע�����ĵ�ַ��Ҳ����Eureka-Server���õĵ�ַ

2.3���������@EnableDiscoveryClientע�ͣ�ʵ�ַ���ע��ͷ���

```
@EnableDiscoveryClient
@EnableConfigServer
@SpringBootApplication
public class ConfigServer {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServer.class, args);
    }
}
```

### 3.����Client�ˣ��������ķ���

3.1Eureka-Client����

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

3.2���������ļ�

```
#spring.cloud.config.uri= http://localhost:8888/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
eureka.client.serviceUrl.defaultZone=http://localhost:8880/eureka/
```

ע�͵������Server�˵�ַ  
spring.cloud.config.discovery.enabled������������֧��  
spring.cloud.config.discovery.serviceId�������ṩ�˵�����  
eureka.client.serviceUrl.defaultZone���������ĵĵ�ַ

### 4.����

��������ע������eurekaServer���˿�Ϊ8880��Ȼ���������config-server�ˣ��˿ڷֱ�Ϊ��8887��8888������������config-client�ˣ��˿ڷֱ��ǣ�8883��8884��  
���Բ鿴ע�����ģ�ע��ķ���  
![](https://oscimg.oschina.net/oscnet/d5f65edf21bc09fd754e85674196875fbcc.jpg)

�ֱ����[http://localhost](http://localhost/):8883/hello��[http://localhost](http://localhost/):8884/hello��������£�

```
hello test
```

����git�е������ļ�Ϊ��

```
foo=hello test update
```

POST��ʽ����Server�ˣ��������������ļ�

```
c:\curl-7.61.0\I386>curl -X POST http://localhost:8888/actuator/bus-refresh
```

����ֻ��ѡ��������һ��server��ȥ���£�����һ�������ԣ�

�ֱ����[http://localhost](http://localhost/):8883/hello��[http://localhost](http://localhost/):8884/hello��������£�

```
hello test update
```

2���ͻ��˶���ȡ�������µ����ݣ���ʾ���³ɹ���

��8888�˿ڵ�Server��ͣ�����ٴθ��������ļ�Ϊ

```
foo=hello test update2
```

POST��ʽ����Server�ˣ��������������ļ�

```
c:\curl-7.61.0\I386>curl -X POST http://localhost:8887/actuator/bus-refresh
```

�ֱ����[http://localhost](http://localhost/):8883/hello��[http://localhost](http://localhost/):8884/hello��������£�

```
hello test update2
```

2���ͻ��˶���ȡ�������µ����ݣ���ʾ���³ɹ���

## **�ܽ�**

ͨ����Ϣ���ߵķ�ʽ����˶��Client���µ����⣬�Լ�ͨ��eureka����֤Server�ĸ߿����ԣ���Ȼeurekaע�����ĺ���Ϣ���߱���Ҳ��Ҫ�߿����ԣ�����Ͳ���������ˡ�

## **ʾ�������ַ**

Github:[https://github.com/ksfzhaohui��](https://github.com/ksfzhaohui/blog)