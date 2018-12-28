**简介**  
Hessian是一个轻量级的remoting onhttp工具，使用简单的方法提供了RMI的功能；相比WebService，Hessian更简单、快捷。  
官网地址：[http://hessian.caucho.com/index.xtp](http://hessian.caucho.com/index.xtp)  
下面主要针对Hessian入门级使用，以及进行部门源码分析。

**简单使用**  
提供三个模拟块，分别模拟client，server以及被依赖的jar；对应的模块名称分别是：hessianClient，hessianServer以及hessianJar  
代码结构如下图所示:

![](https://static.oschina.net/uploads/space/2017/0507/160447_zMBC_159239.png)

**1.hessianJar介绍**  
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

**2.hessianSever介绍**  
hessianServer主要用来对外提供服务的，因为hessian本身是基于http协议的，所以可以直接部署到web容器中比如tomcat，http协议解析可以直接交给web容器；当然这里hessianServer本身是一个web项目，除了对外提供的服务类HessianServiceImpl，还需要配置web.xml文件。  
web.xml：

```
<web-app>
    <display-name>Archetype Created Web Application</display-name>
 
    <servlet>
        <servlet-name>hessianService</servlet-name>
        <servlet-class>com.caucho.hessian.server.HessianServlet</servlet-class>
        <init-param>
            <param-name>home-class</param-name>
            <param-value>zh.hessian.hessianServer.HessianServiceImpl</param-value>
        </init-param>
        <init-param>
            <param-name>home-api</param-name>
            <param-value>zh.hessian.hessianJar.IHessianService</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>hessianService</servlet-name>
        <url-pattern>/hessianService</url-pattern>
    </servlet-mapping>
</web-app>
```

服务类HessianServiceImpl:

```
public class HessianServiceImpl implements IHessianService {
 
    @Override
    public String getString(String value) {
        return "REQ + " + value;
    }
 
    @Override
    public Bean getBean() {
        return new Bean("value");
    }
}
```

**3.hessianClient介绍**  
hessianClient模拟客户端的调用，对hessianSever发起请求，并且接受回复；提供HessianClient类  
HessianClient：

```
public class HessianClient {
 
    public static void main(String[] args) {
        String url = "http://localhost:8080/hessianServer/hessianService";
        HessianProxyFactory factory = new HessianProxyFactory();
        try {
            IHessianService hessianService = (IHessianService) factory.create(IHessianService.class, url);
            System.out.println("getString:" + hessianService.getString("zhaohui"));
            Bean bean = hessianService.getBean();
            System.out.println("getBean:" + bean.getValue());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**部署测试**  
1.首先将hessianJar打包安装，maven命令：clean install  
2.将hessianServer打包发布，maven命令：clean package，然后部署到tomcat中，启动tomcat  
3.将hessianClient打包运行，maven命令：clean package，然后运行HessianClient类

结果如下：

```
getString:REQ + zhaohui
getBean:value
```

**hessianClient流程分析**  
**_1.代理类HessianProxy_**  
很多开源的RPC框架，用来实现远程方法调用，其实核心思想都是动态代理，代理类封装了对消息的序列化，远程的连接以及消息的接受，Hessian也不列外，提供了HessianProxy类，部分代码如下：

```
public class HessianProxy implements InvocationHandler, Serializable {
   public Object invoke(Object proxy, Method method, Object []args)
    throws Throwable
    {
      ......
      conn = sendRequest(mangleName, args);
      is = getInputStream(conn);
      ......
    }
    .....
}
```

HessianProxy实现了invoke方法，这样每次在调用方法(如：hessianService.getString(“zhaohui”))时，自动触发invoke方法，这样我们就可以封装方法和请求参数，建立http请求(sendRequest方法)，接受响应(getInputStream方法)。

_**2.http请求类**_  
既然Hessian本身是基于http协议的，对http的请求我们会想到HttpURLConnection类，Hessian也不例外，只是被包装成了HessianConnection类，代码结构如下：

```
public interface HessianConnection {
  /**
   * Adds HTTP headers.
   */
  public void addHeader(String key, String value);
   
  /**
   * Returns the output stream for the request.
   */
  public OutputStream getOutputStream()
    throws IOException;
 
  /**
   * Sends the query
   */
  public void sendRequest()
    throws IOException;
 
  /**
   * Returns the status code.
   */
  public int getStatusCode();
 
  /**
   * Returns the status string.
   */
  public String getStatusMessage();
   
  /**
   * Returns the content encoding
   */
  public String getContentEncoding();
   
 
  /**
   * Returns the InputStream to the result
   */
  public InputStream getInputStream()
    throws IOException;
 
  /**
   * Close/free the connection. If keepalive is allowed, it may be used.
   */
  public void close()
    throws IOException;
 
  /**
   * Shut the connection down.
   */
  public void destroy()
    throws IOException;
}
```

里面提供了一些常用的方法，添加http头文件addHeader()，获取输出流getOutputStream()，获取输入流getInputStream()等；

_**3.发送请求**_  
有了HessianConnection就可以发送http请求了，HessianProxy中提供了sendRequest方法，部分代码如下：

```
protected HessianConnection sendRequest(String methodName, Object []args)
    throws IOException
  {
    HessianConnection conn = null;
    conn = _factory.getConnectionFactory().open(_url);
    boolean isValid = false;
    try {
      addRequestHeaders(conn);
      OutputStream os = null;
      try {
        os = conn.getOutputStream();
      } catch (Exception e) {
        throw new HessianRuntimeException(e);
      }
      ......
      AbstractHessianOutput out = _factory.getHessianOutput(os);
 
      out.call(methodName, args);
      out.flush();
```

首先在请求中添加了http请求头分别是：Content-Type，Accept-Encoding以及Authorization，如下所示：

```
protected void addRequestHeaders(HessianConnection conn)
{
  conn.addHeader("Content-Type", "x-application/hessian");
  conn.addHeader("Accept-Encoding", "deflate");
 
  String basicAuth = _factory.getBasicAuth();
  if (basicAuth != null)
    conn.addHeader("Authorization", basicAuth);
}
```

然后通过OutputStream向服务器发送请求，Hessian把OutputStream封装在HessianOutput中，所以可以查看HessianOutput中call方法，部分代码如下：

```
public void call(String method, Object []args)
  throws IOException
{
  int length = args != null ? args.length : 0;
   
  startCall(method, length);
   
  for (int i = 0; i < length; i++)
    writeObject(args[i]);
   
  completeCall();
}
```

首先是写入方法名称，如何写入方法参数，最后完成写入标识，

(1).startCall方法如下：

```
public void startCall(String method, int length)
  throws IOException
{
  os.write('c');
  os.write(_version);
  os.write(0);
 
  os.write('m');
  int len = method.length();
  os.write(len >> 8);
  os.write(len);
  printString(method, 0, len);
}
```

前三个自己分别写入了c major minor，后面写入自己m表示开始写入方法信息，包括长度和字符串信息；

(2).写入方法参数，遍历args\[\]使用writeObject写入，主要是对参数进行序列化操作，核心类Serializer，Hessian会根据参数的类型使用不同的序列化，具体有哪些类型可以查看包com.caucho.hessian.io中实现实现了Serializer的类；

(3).最后写入结束标识，写入了字符’z’。

发送请求通过out.flush()方法，至此大概是客户端请求的整个流程，下面介绍一下服务器接受消息，处理消息以及回复消息

_**4.服务器端接受消息**_  
web.xml中配置了处理消息的servlet:HessianServlet，并且配置了真实处理业务逻辑的api和class类，HessianServlet继承了两个核心的方法init和service，分别用来初始化数据和处理业务逻辑。  
init()方法  
init方法中加载了配置在web.xml中的HessianServiceImpl和IHessianService，并且封装到HessianSkeleton中  
service()方法  
通过HttpServletRequest和HttpServletResponse对象分别获取了我们需要的输入和输出流，如下：  
InputStream is = request.getInputStream();  
OutputStream os = response.getOutputStream();  
接下来把InputStream和OutputStream传入HessianSkeleton中，消息的读取和回复都交给HessianSkeleton来处理

HessianSkeleton的invoke方法：

```
public void invoke(InputStream is, OutputStream os,
                     SerializerFactory serializerFactory)
{
    ....
    AbstractHessianInput in;
    AbstractHessianOutput out;
    ...
    in = _hessianFactory.createHessianInput(is);
    out = _hessianFactory.createHessian2Output(os);
    ...
    invoke(_service, in, out);
}
```

又进一步的将Inputstream封装入HessianInput中，将Outputstream封装入Hessian2Output中；  
最下面的invoke(_service, in, out)方法是服务器处理最重要的地方，按照Client中输出流的写入格式，通过Inputstream将Client写入的消息读取出来包括头字节信息，方法名，参数列表，部分代码如下：

```
public void invoke(Object service,
                     AbstractHessianInput in,
                     AbstractHessianOutput out)
    throws Exception
  {
    ServiceContext context = ServiceContext.getContext();
    in.skipOptionalCall();
    ......//省略
    String methodName = in.readMethod();
    int argLength = in.readMethodArgLength();
 
    Method method;
    method = getMethod(methodName + "__" + argLength);
    if (method == null)
      method = getMethod(methodName);
 
    ......//省略
 
    Class<?> []args = method.getParameterTypes();
 
    if (argLength != args.length && argLength >= 0) {
      out.writeFault("NoSuchMethod",
                     escapeMessage("method " + method + " argument length mismatch, received length=" + argLength),
                     null);
      out.close();
      return;
    }
 
    Object []values = new Object[args.length];
 
    for (int i = 0; i < args.length; i++) {
      // XXX: needs Marshal object
      values[i] = in.readObject(args[i]);
    }
 
    Object result = null;
 
    try {
      result = method.invoke(service, values);
    } catch (Exception e) {
      Throwable e1 = e;
      if (e1 instanceof InvocationTargetException)
        e1 = ((InvocationTargetException) e).getTargetException();
 
      log.log(Level.FINE, this + " " + e1.toString(), e1);
 
      out.writeFault("ServiceException", 
                     escapeMessage(e1.getMessage()), 
                     e1);
      out.close();
      return;
    }
 
    // The complete call needs to be after the invoke to handle a
    // trailing InputStream
    in.completeCall();
 
    out.writeReply(result);
 
    out.close();
  }
```

(1).首先通过方法skipOptionalCall()跳过了输入流中的前几个字节，接下来获取了MethodName，还有方法参数存放在了Object \[\]values中；  
(2).接下来就是result = method.invoke(service, values);反射调用，获取到我们需要的result  
(3).out.writeReply(result)，通过Hessian2Output回复消息，代码如下：

```
public void writeReply(Object o)
  throws IOException
{
  startReply();
  writeObject(o);
  completeReply();
}
```

回复消息也指定了对应的格式，消息头+消息体+消息结束标识，具体内容可以自行去查看。

_**5.Client接受服务器的回复**_  
再次回到Client端，在介绍HessianProxy类的时候提到了两个主要的方法分别是：sendRequest发送消息，getInputStream获取输入流，通过输入流来获取服务器端的回复。  
同样的Inputstream也被封装成了Hessian2Input对象，通过Hessian2Input的readReply获取回复，和服务器读取客户端消息类似，此处不在详细介绍，可以直接查看readReply方法

**总结**  
本文通过hessianJar，hessianClient已经hessianServer三个模块，提供了简单的入门实例；然后从代码层面介绍了整个消息的流程，从创建代理类HessianProxy到提供http连接的HessianConnection，然后发送http请求到服务器端接受http请求，反射调用业务逻辑类获取处理结果，然后封装消息回复Client，最后Client接受消息，输出消息；当然代码层面的介绍还是比较笼统的，很多细节还需要进一步了解。

**个人博客：[codingo.xyz](http://codingo.xyz/)**