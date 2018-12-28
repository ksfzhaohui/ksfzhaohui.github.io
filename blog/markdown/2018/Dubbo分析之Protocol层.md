##ϵ������
[Dubbo����Serialize��][1]
[Dubbo����֮Transport��][2]
[Dubbo����֮Exchange ��][3]
[Dubbo����֮Protocol��][4]

##ǰ��
����������[Dubbo����֮Exchange��][5]����������protocolԶ�̵��ò㣬�ٷ����ܣ���װRPC���ã���Invocation, ResultΪ���ģ���չ�ӿ�ΪProtocol, Invoker, Exporter��

##Protocol�ӿ������
Protocol����˵��Dubbo�ĺ��Ĳ��ˣ��ڴ˻����Ͽ�����չ�ܶ������ķ��񣬱��磺redis��Memcached��rmi��WebService��http(tomcat��jetty)�ȵȣ����濴һ�½ӿ���Դ�룺

```
public interface Protocol {
    /**
     * ��¶Զ�̷���<br>
     * 1. Э���ڽ�������ʱ��Ӧ��¼������Դ����ַ��Ϣ��RpcContext.getContext().setRemoteAddress();<br>
     * 2. export()�������ݵȵģ�Ҳ���Ǳ�¶ͬһ��URL��Invoker���Σ��ͱ�¶һ��û������<br>
     * 3. export()�����Invoker�ɿ��ʵ�ֲ����룬Э�鲻��Ҫ���ġ�<br>
     * 
     * @param <T> ���������
     * @param invoker �����ִ����
     * @return exporter ��¶��������ã�����ȡ����¶
     * @throws RpcException ����¶�������ʱ�׳�������˿���ռ��
     */
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
  
    /**
     * ����Զ�̷���<br>
     * 1. ���û�����refer()�����ص�Invoker�����invoke()����ʱ��Э������Ӧִ��ͬURLԶ��export()�����Invoker�����invoke()������<br>
     * 2. refer()���ص�Invoker��Э��ʵ�֣�Э��ͨ����Ҫ�ڴ�Invoker�з���Զ������<br>
     * 3. ��url��������check=falseʱ������ʧ�ܲ����׳��쳣�����ڲ��Զ��ָ���<br>
     * 
     * @param <T> ���������
     * @param type ���������
     * @param url Զ�̷����URL��ַ
     * @return invoker ����ı��ش���
     * @throws RpcException �����ӷ����ṩ��ʧ��ʱ�׳�
     */
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
  
}
```
��Ҫ������2���ӿڣ�һ���Ǳ�¶Զ�̷�����һ��������Զ�̷�����ʵ���Ƿ���˺Ϳͻ��ˣ�dubbo�ṩ�˶Զ��ַ������չ�����Բ鿴META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol��

```
filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=com.alibaba.dubbo.rpc.support.MockProtocol
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
injvm=com.alibaba.dubbo.rpc.protocol.injvm.InjvmProtocol
rmi=com.alibaba.dubbo.rpc.protocol.rmi.RmiProtocol
hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol
com.alibaba.dubbo.rpc.protocol.http.HttpProtocol
com.alibaba.dubbo.rpc.protocol.webservice.WebServiceProtocol
thrift=com.alibaba.dubbo.rpc.protocol.thrift.ThriftProtocol
memcached=com.alibaba.dubbo.rpc.protocol.memcached.MemcachedProtocol
redis=com.alibaba.dubbo.rpc.protocol.redis.RedisProtocol
rest=com.alibaba.dubbo.rpc.protocol.rest.RestProtocol
registry=com.alibaba.dubbo.registry.integration.RegistryProtocol
qos=com.alibaba.dubbo.qos.protocol.QosProtocolWrapper
```
dubboЭ����Ĭ���ṩ��Э�飬������չ��Э�������hessian��http(tomcat��jetty)��injvm��memcached��redis��rest��rmi��thrift��webservice��������չ��Э����Щ��������Ϊ����Զ�̷������(�ͻ���)������redis��memcached��ͨ���ض�������Ի�����в�������ȻҲ������չ�Լ���Э�飬�ֱ�ʵ�ֽӿ���Protocol, Invoker, Exporter��֮ǰ�ֱ���ܵ�serialize�㣬transport���Լ�exchange����Ҫ����ʹ��Ĭ�ϵ�DubboProtocol�������⼸���ײ㣬������չЭ��ֱ��������������չ����
�����ص����һ��DubboProtocol�࣬���ȿ�һ��referʵ�ַ�����

```
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // create rpc invoker.
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```
�ڿͻ��˶�һ����ÿ��dubbo:reference�������ڴ˴�ʵ����һ����Ӧ��DubboInvoker���ڷ����ڲ����ȶ����л��Ż����д�����Ҫ�Ƕ�Kryo,FST�����л���ʽ�����Ż����˷��������ڿͻ��ˣ�ͬʱ��������Ҳ���ڣ����������Ǵ�����һ��DubboInvoker��ͬʱ������������˵����ӣ�

```
private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        boolean service_share_connect = false;
        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
        // if not configured, connection is shared, otherwise, one connection for one service
        if (connections == 0) {
            service_share_connect = true;
            connections = 1;
        }
 
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (service_share_connect) {
                clients[i] = getSharedClient(url);
            } else {
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
```
Ĭ����ָ���ķ���������һ�����ӣ�����ͨ��ָ��connections���ý���������ӣ��ڲ����Ƚϴ������¿������ö����

```
private ExchangeClient initClient(URL url) {
 
        // client type setting.
        String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));
 
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
 
        // BIO is not allowed since it has severe performance issue.
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }
 
        ExchangeClient client;
        try {
            // connection should be lazy
            if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
                client = new LazyConnectExchangeClient(url, requestHandler);
            } else {
                client = Exchangers.connect(url, requestHandler);
            }
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
        return client;
    }
```
�˷�����Ҫͨ��Exchange��ӿ����ͷ���˽������ӣ�ͬʱ�ṩ�������ӵķ�ʽ��Ҫ�ȵ��������������ʱ��Ž������ӣ�����ExchangeClient��DubboInvoker�ڲ�ͨ��ExchangeClient���������������ˣ�������һ��export������

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
       URL url = invoker.getUrl();
 
       // export service.
       String key = serviceKey(url);
       DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
       exporterMap.put(key, exporter);
 
       //export an stub service for dispatching event
       Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
       Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
       if (isStubSupportEvent && !isCallbackservice) {
           String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
           if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
               if (logger.isWarnEnabled()) {
                   logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                           "], has set stubproxy support event ,but no stub methods founded."));
               }
           } else {
               stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
           }
       }
 
       openServer(url);
       optimizeSerialization(url);
       return exporter;
   }
```
ÿ��dubbo:service�����һ��Exporter������ͨ��url��ȡһ��key(������port��serviceName��serviceVersion��serviceGroup)��Ȼ��ʵ������DubboExporterͨ��keyֵ������һ��Map�У������ڽ��յ���Ϣ��ʱ����¶�λ�������Exporter�����������Ǵ�����������

```
private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
        if (isServer) {
            ExchangeServer server = serverMap.get(key);
            if (server == null) {
                serverMap.put(key, createServer(url));
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
    }
 
    private ExchangeServer createServer(URL url) {
        // send readonly event when server closes, it's enabled by default
        url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
        String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
 
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
 
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        str = url.getParameter(Constants.CLIENT_KEY);
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }
        return server;
    }
```
������Ҫ����ͨ��Exchangers��bind�����������������������ض�Ӧ��ExchangeServer��ͬ��Ҳ�����ڱ��ص�Map�У����ͬ���������л��Ż�����

##Invoker�����
refer()���ص�Invoker��Э��ʵ�֣�Э��ͨ����Ҫ�ڴ�Invoker�з���Զ������export()�����Invoker�ɿ��ʵ�ֲ����룬Э�鲻��Ҫ���ģ��ӿ������£�

```
public interface Invoker<T> extends Node {
 
    Class<T> getInterface();
 
    Result invoke(Invocation invocation) throws RpcException;
}
```
���ڽ��ܵ���refer�������ص�Invoker��Ĭ�ϵ�dubboЭ���£�ʵ����DubboInvoker��ʵ�������е�invoke�������˷����ڿͻ��˵���Զ�̷�����ʱ��ᱻ���ã�

```
public Result invoke(Invocation inv) throws RpcException {
    if (destroyed.get()) {
        throw new RpcException("Rpc invoker for service " + this + " on consumer " + NetUtils.getLocalHost()
                + " use dubbo version " + Version.getVersion()
                + " is DESTROYED, can not be invoked any more!");
    }
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (attachment != null && attachment.size() > 0) {
        invocation.addAttachmentsIfAbsent(attachment);
    }
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null) {
        /**
         * invocation.addAttachmentsIfAbsent(context){@link RpcInvocation#addAttachmentsIfAbsent(Map)}should not be used here,
         * because the {@link RpcContext#setAttachment(String, String)} is passed in the Filter when the call is triggered
         * by the built-in retry mechanism of the Dubbo. The attachment to update RpcContext will no longer work, which is
         * a mistake in most cases (for example, through Filter to RpcContext output traceId and spanId and other information).
         */
        invocation.addAttachments(contextAttachments);
    }
    if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) {
        invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
 
 
    try {
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        Throwable te = e.getTargetException();
        if (te == null) {
            return new RpcResult(e);
        } else {
            if (te instanceof RpcException) {
                ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
            }
            return new RpcResult(te);
        }
    } catch (RpcException e) {
        if (e.isBiz()) {
            return new RpcResult(e);
        } else {
            throw e;
        }
    } catch (Throwable e) {
        return new RpcResult(e);
    }
}
 
protected abstract Result doInvoke(Invocation invocation) throws Throwable;
```
��DubboInvoker�ĳ��������ṩ��invoke��������ͳһ�ĸ���(Attachment)������������Ĳ�����һ��RpcInvocation���󣬰����˷������õ���ز�����

```
public class RpcInvocation implements Invocation, Serializable {
 
    private static final long serialVersionUID = -4355285085441097045L;
 
    private String methodName;
 
    private Class<?>[] parameterTypes;
 
    private Object[] arguments;
 
    private Map<String, String> attachments;
 
    private transient Invoker<?> invoker;
     
    ....ʡ��...
}
```
�����˷������ƣ���������������ֵ��������Ϣ��������ᷢ��û�нӿڣ��汾����Ϣ����Щ��Ϣ��ʵ�����ڸ����У���invoke���������ȴ���ľ��ǰ�attachment��Ϣ���浽RpcInvocation�У����������ǵ���DubboInvoker�е�doInvoke������

```
protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
 
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                ResponseFuture future = currentClient.request(inv, timeout);
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
�˷������Ȼ�ȡExchangeClient�����ʵ�����˶��ExchangeClient����ͨ��˳��ķ�ʽ����ʹ��ExchangeClient��ͨ��ExchangeClient��RpcInvocation���͸��������ˣ��ṩ�����ַ��ͷ�ʽ������ͨ�ŷ�ʽ��˫��ͨ��(ͬ��)��˫��ͨ��(�첽)��������[Dubbo����֮Exchange��][6]�У�����������֮��ֱ�ӷ���DefaultFuture�������������get����������ֱ�����ؽ�����߳�ʱ��ͬ����ʽ����ֱ�ӵ���get�����������ȴ�����������ص㿴һ���첽��ʽ���첽��ʽ�����ص�DefaultFuture������RpcContext�У�Ȼ�󷵻���һ���ն���������ʵʹ����ThreadLocal���ܣ�����ÿ���ڿͻ���ҵ������У��������첽���󣬶���Ҫͨ��RpcContext��ȡResponseFuture�����磺

```
// �˵��û���������null
fooService.findFoo(fooId);
// �õ����õ�Future���ã���������غ󣬻ᱻ֪ͨ�����õ���Future
Future<Foo> fooFuture = RpcContext.getContext().getFuture(); 
  
// �˵��û���������null
barService.findBar(barId);
// �õ����õ�Future���ã���������غ󣬻ᱻ֪ͨ�����õ���Future
Future<Bar> barFuture = RpcContext.getContext().getFuture(); 
  
// ��ʱfindFoo��findBar������ͬʱ��ִ�У��ͻ��˲���Ҫ�������߳���֧�ֲ��У����ǽ���NIO�ķ��������
  
// ���foo�ѷ��أ�ֱ���õ�����ֵ�������߳�waitס���ȴ�foo���غ��̻߳ᱻnotify����
Foo foo = fooFuture.get(); 
// ͬ��ȴ�bar����
Bar bar = barFuture.get(); 
  
// ���foo��Ҫ5�뷵�أ�bar��Ҫ6�뷵�أ�ʵ��ֻ���6�룬���ɻ�ȡ��foo��bar�����н������Ĵ���
```
������һ�����ӣ��ܺõ�˵�����첽��ʹ�÷�ʽ�Լ������ƣ�

##Exporter�����
������[Dubbo����֮Exchange��][7]�У�����˽��յ���Ϣ֮�󣬵���handler��reply����������Ϣ������handler������DubboProtocol�У����£�

```
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
 
        @Override
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                Invoker<?> invoker = getInvoker(channel, inv);
                // need to consider backward-compatibility if it's a callback
                if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                    String methodsStr = invoker.getUrl().getParameters().get("methods");
                    boolean hasMethod = false;
                    if (methodsStr == null || methodsStr.indexOf(",") == -1) {
                        hasMethod = inv.getMethodName().equals(methodsStr);
                    } else {
                        String[] methods = methodsStr.split(",");
                        for (String method : methods) {
                            if (inv.getMethodName().equals(method)) {
                                hasMethod = true;
                                break;
                            }
                        }
                    }
                    if (!hasMethod) {
                        logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                                + " not found in callback service interface ,invoke will be ignored."
                                + " please update the api interface. url is:"
                                + invoker.getUrl()) + " ,invocation is :" + inv);
                        return null;
                    }
                }
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: "
                    + (message == null ? null : (message.getClass().getName() + ": " + message))
                    + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }
         
        ...ʡ��...
}
```
����˽��յ�message���������RpcInvocation����������˽ӿڣ���������������Ϣ����������ͨ������ķ�ʽ���������Ȼ�ȡ�˶�Ӧ��DubboExporter�������ȡ��ͨ��key(������port��serviceName��serviceVersion��serviceGroup)��ȡ��Ӧ��DubboExporter��Ȼ�����DubboExporter�е�invoker����ʱ��invoker��ϵͳ�������ģ�����ͻ���Invoker��Э����Լ������ģ�ϵͳ������invoker��������ķ�ʽ���ڣ��ڲ����ö�Ӧ��filter����������Щfilter������������ʱ�Ѿ���ʼ��������ProtocolFilterWrapper��buildInvokerChain�У���������Щfilter���Բ鿴META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Filter:

```
cache=com.alibaba.dubbo.cache.filter.CacheFilter
validation=com.alibaba.dubbo.validation.filter.ValidationFilter
echo=com.alibaba.dubbo.rpc.filter.EchoFilter
generic=com.alibaba.dubbo.rpc.filter.GenericFilter
genericimpl=com.alibaba.dubbo.rpc.filter.GenericImplFilter
token=com.alibaba.dubbo.rpc.filter.TokenFilter
accesslog=com.alibaba.dubbo.rpc.filter.AccessLogFilter
activelimit=com.alibaba.dubbo.rpc.filter.ActiveLimitFilter
classloader=com.alibaba.dubbo.rpc.filter.ClassLoaderFilter
context=com.alibaba.dubbo.rpc.filter.ContextFilter
consumercontext=com.alibaba.dubbo.rpc.filter.ConsumerContextFilter
exception=com.alibaba.dubbo.rpc.filter.ExceptionFilter
executelimit=com.alibaba.dubbo.rpc.filter.ExecuteLimitFilter
deprecated=com.alibaba.dubbo.rpc.filter.DeprecatedFilter
compatible=com.alibaba.dubbo.rpc.filter.CompatibleFilter
timeout=com.alibaba.dubbo.rpc.filter.TimeoutFilter
trace=com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter
future=com.alibaba.dubbo.rpc.protocol.dubbo.filter.FutureFilter
monitor=com.alibaba.dubbo.monitor.support.MonitorFilter
```
�����г������е�filter���������Ѷ˺ͷ���ˣ�����ʹ����Щ��ͨ��filter��ע��@Activate�����й��ˣ�ÿ��filter�����˷��飻����ִ�е�˳������ô���ģ�ͬ����ע������ָ���ˣ���ʽ���£�

```
@Activate(group = Constants.PROVIDER, order = -110000)
@Activate(group = Constants.PROVIDER, order = -10000)
@Activate(group = Constants.CONSUMER, value = Constants.GENERIC_KEY, order = 20000)
```
ÿ���̶���filter�и��ԵĹ��ܣ�ͬ��Ҳ���Խ�����չ���������˽�����һ�������ͨ��������÷���RpcResult��

##�ܽ�
���Ĵ��������һ��Protocol��ʹ�õ�Ĭ��dubboЭ����ܣ�Protocol�㻹������������Э���������չ�������������ܣ��������filter����������ϸ����һ�£�

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][8]
[https://gitee.com/OutOfMemory/blog][9]


  [1]: https://segmentfault.com/a/1190000016620859
  [2]: https://segmentfault.com/a/1190000016777060
  [3]: https://segmentfault.com/a/1190000016802475
  [4]: https://segmentfault.com/a/1190000016885261
  [5]: https://segmentfault.com/a/1190000016802475
  [6]: https://segmentfault.com/a/1190000016802475
  [7]: https://segmentfault.com/a/1190000016802475
  [8]: https://github.com/ksfzhaohui/blog
  [9]: https://gitee.com/OutOfMemory/blog