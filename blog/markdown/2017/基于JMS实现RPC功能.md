**前言**  
JMS的点对点消息传送模式允许JMS客户端通过队列(queue)这个虚拟通道来同步和异步发送、接收消息；点对点消息传送模式既支持异步“即发即弃”消息传送方式，又支持同步请求/应答消息传送方式；点对点消息传送模式支持负载均衡，它允许多个接收者监听同一队列，并以此来分配负载；所以完全可以以JMS的点对点消息传送模式实现一套RPC框架。

**准备**  
jdk：jdk1.7.0_80  
jms：apache-activemq-5.10.0  
serialize：protostuff

**整体结构**  
整个实现包括：rpc-jms-common，rpc-jms-client，rpc-jms-server；  
以及测试模块：rpc-jms-test-api，rpc-jms-test-client，rpc-jms-test-server  
整体如下图所示：

![](https://static.oschina.net/uploads/space/2017/0613/222533_F2es_159239.png)

rpc-jms-common：公共模块  
rpc-jms-client：给客户端使用的支持库  
rpc-jms-server：给服务端使用的支持库

**客户端实现**  
客户端其实核心思想都是动态代理，生成的代理类将接口名称，版本信息，方法名，参数类型，参数值等信息封装成RpcRequest，然后使用protostuff对其进行序列化操作，将序列化后的二进制数据放入BytesMessage中，发送给消息队列；然后等待消息的返回，从BytesMessage中获取二进制数据，使用protostuff进行反序列化操作；最后将结果或者异常展示给用户，具体代码如下：

```
public class RpcClient {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(RpcClient.class);
 
    private QueueConnection qConnection;
    private QueueSession qSession;
    private Queue requestQ;
    private Queue responseQ;
 
    public RpcClient(String rpcFactory, String rpcRequest, String rpcResponse) {
        try {
            Context ctx = new InitialContext();
            QueueConnectionFactory factory = (QueueConnectionFactory) ctx.lookup(rpcFactory);
            qConnection = factory.createQueueConnection();
            qSession = qConnection.createQueueSession(false, Session.AUTO_ACKNOWLEDGE);
            requestQ = (Queue) ctx.lookup(rpcRequest);
            responseQ = (Queue) ctx.lookup(rpcResponse);
            qConnection.start();
        } catch (Exception e) {
            LOGGER.error("init rpcproxy error", e);
        }
    }
 
    public <T> T create(final Class<?> interfaceClass) {
        return create(interfaceClass, "");
    }
 
    @SuppressWarnings("unchecked")
    public <T> T create(final Class<?> interfaceClass, final String serviceVersion) {
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[] { interfaceClass },
                new InvocationHandler() {
 
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        RpcRequest request = new RpcRequest();
                        request.setRequestId(UUID.randomUUID().toString());
                        request.setInterfaceName(method.getDeclaringClass().getName());
                        request.setServiceVersion(serviceVersion);
                        request.setMethodName(method.getName());
                        request.setParameterTypes(method.getParameterTypes());
                        request.setParameters(args);
 
                        BytesMessage requestMessage = qSession.createBytesMessage();
                        requestMessage.writeBytes(SerializationUtil.serialize(request));
                        requestMessage.setJMSReplyTo(responseQ);
                        QueueSender qsender = qSession.createSender(requestQ);
                        qsender.send(requestMessage);
 
                        String filter = "JMSCorrelationID = '" + requestMessage.getJMSMessageID() + "'";
                        QueueReceiver qReceiver = qSession.createReceiver(responseQ, filter);
                        BytesMessage responseMessage = (BytesMessage) qReceiver.receive(30000);
                        byte messByte[] = new byte[(int) responseMessage.getBodyLength()];
                        responseMessage.readBytes(messByte);
                        RpcResponse rpcResponse = SerializationUtil.deserialize(messByte, RpcResponse.class);
 
                        if (rpcResponse.hasException()) {
                            throw rpcResponse.getException();
                        } else {
                            return rpcResponse.getResult();
                        }
                    }
                });
    }
}
```

RpcClient被创建的时候就和jms建立了连接，相关jms的配置信息在测试部分讲解。

其中封装请求类RpcRequest代码如下：

```
public class RpcRequest {
 
    private String requestId;
    private String interfaceName;
    private String serviceVersion;
    private String methodName;
    private Class<?>[] parameterTypes;
    private Object[] parameters;
    //省略get和set方法
}
```

封装响应类RpcResponse代码如下：

```
public class RpcResponse{
 
    private String requestId;
    private Exception exception;
    private Object result;
    //省略get和set方法
}
```

**服务端实现**  
服务器端首先加载所有需要对外提供服务的类对象；然后监听消息队列，从BytesMessage中获取二进制数据，通过protostuff反序列化为RpcRequest对象；最后通过反射的方式调用服务类，获取的结果封装成RpcResponse，然后序列化二进制写入BytesMessage中，发送给客户端，具体代码如下：

```
public class RpcServer implements ApplicationContextAware, InitializingBean {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(RpcServer.class);
 
    private QueueConnection qConnection;
    private QueueSession qSession;
    private Queue requestQ;
 
    private String rpcFactory;
    private String rpcRequest;
 
    /**
     * 存放 服务名 与 服务对象 之间的映射关系
     */
    private Map<String, Object> serviceMap = new HashMap<String, Object>();
 
    public RpcServer(String rpcFactory, String rpcRequest) {
        this.rpcFactory = rpcFactory;
        this.rpcRequest = rpcRequest;
    }
 
    @Override
    public void afterPropertiesSet() throws Exception {
        try {
            Context ctx = new InitialContext();
            QueueConnectionFactory factory = (QueueConnectionFactory) ctx.lookup(rpcFactory);
            qConnection = factory.createQueueConnection();
            qSession = qConnection.createQueueSession(false, Session.AUTO_ACKNOWLEDGE);
            requestQ = (Queue) ctx.lookup(rpcRequest);
            qConnection.start();
 
            QueueReceiver receiver = qSession.createReceiver(requestQ);
            receiver.setMessageListener(new RpcMessageListener(qSession, serviceMap));
 
            LOGGER.info("ready receiver message");
        } catch (Exception e) {
            if (qConnection != null) {
                try {
                    qConnection.close();
                } catch (JMSException e1) {
                }
            }
            LOGGER.error("server start error", e);
        }
    }
 
    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        Map<String, Object> serviceBeanMap = ctx.getBeansWithAnnotation(RpcService.class);
        if (serviceBeanMap != null && serviceBeanMap.size() > 0) {
            for (Object serviceBean : serviceBeanMap.values()) {
                RpcService rpcService = serviceBean.getClass().getAnnotation(RpcService.class);
                String serviceName = rpcService.value().getName();
                String serviceVersion = rpcService.version();
                if (serviceVersion != null && !serviceVersion.equals("")) {
                    serviceName += "-" + serviceVersion;
                }
                serviceMap.put(serviceName, serviceBean);
            }
        }
    }
 
}
```

消息监听器类RpcMessageListener：

```
public class RpcMessageListener implements MessageListener {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(RpcMessageListener.class);
 
    private QueueSession qSession;
    /**
     * 存放 服务名 与 服务对象 之间的映射关系
     */
    private Map<String, Object> serviceMap = new HashMap<String, Object>();
 
    public RpcMessageListener(QueueSession qSession, Map<String, Object> serviceMap) {
        this.qSession = qSession;
        this.serviceMap = serviceMap;
    }
 
    @Override
    public void onMessage(Message message) {
        try {
            LOGGER.info("receiver message : " + message.getJMSMessageID());
            RpcResponse response = new RpcResponse();
            BytesMessage responeByte = qSession.createBytesMessage();
            responeByte.setJMSCorrelationID(message.getJMSMessageID());
            QueueSender sender = qSession.createSender((Queue) message.getJMSReplyTo());
            try {
                BytesMessage byteMessage = (BytesMessage) message;
                byte messByte[] = new byte[(int) byteMessage.getBodyLength()];
                byteMessage.readBytes(messByte);
                RpcRequest rpcRequest = SerializationUtil.deserialize(messByte, RpcRequest.class);
 
                response.setRequestId(rpcRequest.getRequestId());
 
                String serviceName = rpcRequest.getInterfaceName();
                String serviceVersion = rpcRequest.getServiceVersion();
                if (serviceVersion != null && !serviceVersion.equals("")) {
                    serviceName += "-" + serviceVersion;
                }
                Object serviceBean = serviceMap.get(serviceName);
                if (serviceBean == null) {
                    throw new RuntimeException(String.format("can not find service bean by key: %s", serviceName));
                }
 
                Class<?> serviceClass = serviceBean.getClass();
                String methodName = rpcRequest.getMethodName();
                Class<?>[] parameterTypes = rpcRequest.getParameterTypes();
                Object[] parameters = rpcRequest.getParameters();
 
                Method method = serviceClass.getMethod(methodName, parameterTypes);
                method.setAccessible(true);
                Object result = method.invoke(serviceBean, parameters);
                response.setResult(result);
            } catch (Exception e) {
                response.setException(e);
                LOGGER.error("onMessage error", e);
            }
            responeByte.writeBytes(SerializationUtil.serialize(response));
            sender.send(responeByte);
        } catch (Exception e) {
            LOGGER.error("send message error", e);
        }
    }
 
}
```

**测试**  
1.rpc-jms-test-api：接口模块，被rpc-jms-test-client和rpc-jms-test-server共同使用  
IHelloService类：

```
public interface IHelloService {
 
    String hello(String name);
}
```

2.rpc-jms-test-server：服务器端测试模块，依赖rpc-jms-server  
jms相关信息配置在jndi.properties中，如下所示：

```
java.naming.factory.initial=org.apache.activemq.jndi.ActiveMQInitialContextFactory
java.naming.provider.url=tcp://localhost:61616
java.naming.security.principal=system
java.naming.security.credentials=manager
 
connectionFactoryNames=RPCFactory
 
queue.RPCRequest=jms.RPCRequest
queue.RPCResponse=jms.RPCResponse
```

服务器端主要依赖的spring-server.xml配置文件，主要用于实例化RpcServer

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
 
    <context:component-scan base-package="zh.rpc.jms.test.server.impl" />
 
    <context:property-placeholder location="classpath:jms.properties" />
 
    <bean id="rpcServer" class="zh.rpc.jms.server.RpcServer">
        <constructor-arg name="rpcFactory" value="${rpc.rpc_factory}" />
        <constructor-arg name="rpcRequest" value="${rpc.rpc_request}" />
    </bean>
</beans>
```

IHelloService的具体实现类：

```
@RpcService(IHelloService.class)
public class HelloServiceImpl implements IHelloService {
 
    @Override
    public String hello(String name) {
        return "REQ+" + name;
    }
}
```

3.rpc-jms-test-client：客户端测试模块，依赖rpc-jms-client  
客户端同样需要连接消息队列，所以也提供了配置文件jndi.properties；  
客户端主要依赖的spring-client.xml：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
 
    <context:property-placeholder location="classpath:jms.properties" />
 
    <bean id="rpcProxy" class="zh.rpc.jms.client.RpcClient">
        <constructor-arg name="rpcFactory" value="${rpc.rpc_factory}" />
        <constructor-arg name="rpcRequest" value="${rpc.rpc_request}" />
        <constructor-arg name="rpcResponse" value="${rpc.rpc_response}" />
    </bean>
</beans>
```

客户端测试类：

```
public class ClientTest {
 
    private static ApplicationContext context;
 
    public static void main(String[] args) throws Exception {
        context = new ClassPathXmlApplicationContext("spring-client.xml");
        RpcClient rpcProxy = context.getBean(RpcClient.class);
 
        IHelloService helloService = rpcProxy.create(IHelloService.class);
        String result = helloService.hello("World");
        System.out.println(result);
        System.exit(0);
    }
}
```

4.测试  
首先启动事先准备的activemq，运行bin\\win64\\activemq.bat即可；  
然后启动服务器ServerTest，ready receiver message  
最后运行ClientTest，发送消息验证结果，结果如下：

```
REQ+World
```

以上只列出了部分较为重要的代码，更多详细的可以参考：[https://github.com/ksfzhaohui/rpc-jms.git](https://github.com/ksfzhaohui/rpc-jms.git)