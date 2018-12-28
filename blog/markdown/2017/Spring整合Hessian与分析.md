**前言**  
上一篇文章[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)介绍了hessian的简单入门，并且从源码层面对Hessian的调用流程进行了分析；发现使用原生的Hessian还是比较繁琐的，下面看看Spring与Hessian进行整合并且进行简要分析。

**使用**  
提供三个模拟块，分别模拟client，server以及被依赖的jar；对应的模块名称分别是：hessianClient，hessianServer以及hessianJar  
1.hessianJar介绍  
hessianJar主要提供被hessianClient和hessianServer依赖的公共类，这里主要提供了接口类IHessianService和pojo对象Bean  
IHessianService类：

```
public interface IHessianService {
      
    public String getString(String value);
      
    public Bean getBean();
}
```

对象Bean：

```
public class Bean implements Serializable {
    private static final long serialVersionUID = 1L;
    private String value;
  
    public Bean(String value) {
        this.value = value;
    }
  
    public String getValue() {
        return value;
    }
  
    public void setValue(String value) {
        this.value = value;
    }
}
```

2.hessianSever介绍  
hessianServer主要用来对外提供服务的，因为hessian本身是基于http协议的，所以可以直接部署到web容器中比如tomcat，http协议解析可以直接交给web容器；此处因为将服务的发布交给Spring来处理，提供配置文件如下：  
spring-server-hessian.xml：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx" xmlns:lang="http://www.springframework.org/schema/lang"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd  
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd  
           http://www.springframework.org/schema/lang http://www.springframework.org/schema/lang/spring-lang-2.0.xsd  
           http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd">
 
    <bean id="hessionService" class="zh.hessian.hessianServer.HessianServiceImpl" />  
     
    <bean name="/hessianService.do"
        class="org.springframework.remoting.caucho.HessianServiceExporter">
        <property name="service" ref="hessionService" />
        <property name="serviceInterface" value="zh.hessian.hessianJar.IHessianService" />
    </bean>
</beans> 
```

web.xml：

```
<web-app>
  <display-name>Archetype Created Web Application</display-name>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>
                classpath:spring-hessian.xml
            </param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>*.do</url-pattern>
    </servlet-mapping>
</web-app>
```

HessianServiceImpl：

```
public class HessianServiceImpl implements IHessianService {
 
    @Override
    public String getString() {
        return "string";
    }
 
    @Override
    public Bean getBean() {
        return new Bean("value");
    }
}
```

3.hessianClient介绍  
hessianClient模拟客户端的调用，对hessianSever发起请求，并且接受回复；此处因为将客户端的调用交给Spring来处理，提供配置文件如下：  
spring-client-hessian.xml：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd  
     http://www.springframework.org/schema/context  
     http://www.springframework.org/schema/context/spring-context-3.0.xsd">
 
    <bean id="hessionServiceClient"
        class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
        <property name="serviceUrl"
            value="http://localhost:8080/hessianServer/hessianService.do" />
        <property name="serviceInterface" value="zh.hessian.hessianJar.IHessianService" />
    </bean>
</beans>  
```

HessianSpringClient：

```
public class HessianSpringClient {
 
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-client-hessian.xml");
        IHessianService service = (IHessianService) context.getBean("hessionServiceClient");
        System.out.println(service.getString());
        System.out.println(service.getBean().getValue());
    }
}
```

**部署测试**  
运行结果如下：

```
getString:REQ + zhaohui
getBean:value
```

**Spring整合Hessian调用分析**  
1.HessianProxyFactoryBean类  
配置文件spring-client-hessian.xml中定义的对象class都是HessianProxyFactoryBean类，而我们通过上一篇文章中了解到Hessian通过在客户端使用动态代理的方式来实现RPC，HessianProxyFactoryBean代码如下：

```
public class HessianProxyFactoryBean extends HessianClientInterceptor implements FactoryBean<Object> {
 
    private Object serviceProxy;
 
    @Override
    public void afterPropertiesSet() {
        super.afterPropertiesSet();
        this.serviceProxy = new ProxyFactory(getServiceInterface(), this).getProxy(getBeanClassLoader());
    }
 
    public Object getObject() {
        return this.serviceProxy;
    }
 
    public Class<?> getObjectType() {
        return getServiceInterface();
    }
 
    public boolean isSingleton() {
        return true;
    }
}
```

发现HessianProxyFactoryBean实现了FactoryBean接口，而接口的方法getObject()正是我们在调用context.getBean(“x”)的时候被调用，所以获取的bean其实是serviceProxy，通过ProxyFactory的getProxy方法获取，具体代码如下：

```
public Object getProxy(ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

serviceProxy其实是通过AopProxy获取的代理类，AopProxy有两个实现类分别是：CglibAopProxy和JdkDynamicAopProxy，分别对应的两种动态代理方式，具体使用哪种，通过DefaultAopProxyFactory中如下代码来实现：

```
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) 
    {
        Class targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface()) {
            return new JdkDynamicAopProxy(config);
        }
        return CglibProxyFactory.createCglibProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

这里使用的是JdkDynamicAopProxy，部分代码如下：

```
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
    public Object getProxy(ClassLoader classLoader) {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating JDK dynamic proxy: target source is " + 
                              this.advised.getTargetSource());
        }
        Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
        findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
    }
 
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                ......
                Object retVal;
                ......
                invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
                ......
                return retVal;
        }
 
}
```

JdkDynamicAopProxy实现了InvocationHandler接口，每次在调用方法(如：hessianService.getString(“zhaohui”))时，自动触发invoke方法；这里将需要的参数比如：(target,method,args等)封装到了ReflectiveMethodInvocation中，在ReflectiveMethodInvocation的proceed()方法中又调用了HessianProxyFactoryBean中的invoke()方法，部分代码如下：

```
public Object invoke(MethodInvocation invocation) throws Throwable {
        if (this.hessianProxy == null) {
            throw new IllegalStateException("HessianClientInterceptor is not properly initialized - " +
                    "invoke 'prepare' before attempting any operations");
        }
 
        ClassLoader originalClassLoader = overrideThreadContextClassLoader();
        try {
            return invocation.getMethod().invoke(this.hessianProxy, invocation.getArguments());
        }
        ......
}
```

其中最关注的是hessianProxy对象，这里通过反射的方式调用了hessianProxy对象里面的指定方法(比如：hessianService.getString(“zhaohui”))，hessianProxy对象在初始化HessianProxyFactoryBean的时候就初始化好了，具体代码如下：

```
protected Object createHessianProxy(HessianProxyFactory proxyFactory) throws MalformedURLException {
    Assert.notNull(getServiceInterface(), "'serviceInterface' is required");
    return proxyFactory.create(getServiceInterface(), getServiceUrl());
}
```

这段代码有没有很熟悉，就是上一篇文章[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)中的客户端代码，如下所示：

```
String url = "http://localhost:8080/hessianServer-0.0.1-SNAPSHOT/hessianService";
HessianProxyFactory factory = new HessianProxyFactory();
IHessianService hessianService = null;
hessianService = (IHessianService) factory.create(IHessianService.class, url);
```

这里的getServiceInterface()和getServiceUrl()正是我们在spring-client-hessian.xml为hessionServiceClient配置的两个属性，其实到这里下面的流程就和上一篇文章[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)中完全一样了。

2.代理类HessianProxy  
具体分析同[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)

3.http请求类  
具体分析同[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)

4.发送请求  
具体分析同[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)

5.服务器端接受消息  
web.xml中配置了处理消息的servlet:DispatcherServlet，在启动服务器并初始化DispatcherServlet的时候，加载了配置在spring-server-hessian.xml中的org.springframework.remoting.caucho.HessianServiceExporter；DispatcherServlet只是启动一个映射的作用，真正的处理在  
HessianServiceExporter类中，部分代码如下：

```
public void invoke(InputStream inputStream, OutputStream outputStream) throws Throwable {
    Assert.notNull(this.skeleton, "Hessian exporter has not been initialized");
    doInvoke(this.skeleton, inputStream, outputStream);
}
 
protected void doInvoke(HessianSkeleton skeleton, InputStream inputStream, OutputStream outputStream)
        throws Throwable {
        ......
        AbstractHessianInput in;
        AbstractHessianOutput out;
 
        if (code == 'H') {
            // Hessian 2.0 stream
            major = isToUse.read();
            minor = isToUse.read();
            if (major != 0x02) {
                throw new IOException("Version " + major + "." + minor + " is not understood");
            }
            in = new Hessian2Input(isToUse);
            out = new Hessian2Output(osToUse);
            in.readCall();
        }
        else if (code == 'C') {
            // Hessian 2.0 call... for some reason not handled in HessianServlet!
            isToUse.reset();
            in = new Hessian2Input(isToUse);
            out = new Hessian2Output(osToUse);
            in.readCall();
        }
        else if (code == 'c') {
            // Hessian 1.0 call
            major = isToUse.read();
            minor = isToUse.read();
            in = new HessianInput(isToUse);
            if (major >= 2) {
                out = new Hessian2Output(osToUse);
            }
            else {
                out = new HessianOutput(osToUse);
            }
        }
        ......
        skeleton.invoke(in, out);
        ......
}
```

HessianServiceExporter在实例化的同时也初始化了HessianSkeleton对象；又进一步的将Inputstream封装入HessianInput中，将Outputstream封装入Hessian2Output中；接下来把HessianInput和Hessian2Output传入HessianSkeleton中，消息的读取和回复都交给HessianSkeleton来处理；后面的处理具体分析同[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)中介绍的。

5.Client接受服务器的回复  
具体分析同[Hessian入门体验与分析](https://my.oschina.net/OutOfMemory/blog/894300)

**总结**  
本文通过hessianJar，hessianClient已经hessianServer三个模块，提供了Spring整合Hessian的实例；通过与Spring的整合，简化了开发；然后从代码层面将包裹在Hessian外层的Spring剥离，还原原始的Hessian调用。

**个人博客：[codingo.xyz](http://codingo.xyz/)**