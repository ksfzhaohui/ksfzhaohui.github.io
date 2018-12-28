##ǰ��
��������[Dubbo����֮Cluster��][1]�����ļ�������dubbo��register�㣻�˲��װ�����ַ��ע���뷢�֣��Է���URLΪ���ģ���չ�ӿ�ΪRegistryFactory, Registry, RegistryService��

##Registry�ӿ�
�ӿڶ������£�

```
public interface Registry extends Node, RegistryService {
}
 
public interface RegistryService {
 
    void register(URL url);
 
    void unregister(URL url);
     
    void subscribe(URL url, NotifyListener listener);
 
    void unsubscribe(URL url, NotifyListener listener);
 
    List<URL> lookup(URL url);
 
}
```
��Ҫ�ṩ��ע��(register)��ע��(unregister)������(subscribe)���˶�(unsubscribe)�ȹ��ܣ�dubbo�ṩ�˶���ע�᷽ʽ�ֱ��ǣ�Multicast ��Zookeeper��Redis�Լ�Simple��ʽ��
Multicast��Multicastע�����Ĳ���Ҫ�����κ����Ľڵ㣬ֻҪ�㲥��ַһ�����Ϳ��Ի��෢�֣�
Zookeeper��Zookeeper��Apacahe Hadoop������Ŀ����һ�����͵�Ŀ¼����֧�ֱ�����ͣ��ʺ���ΪDubbo�����ע�����ģ���ҵǿ�Ƚϸߣ��������������������Ƽ�ʹ�ã�
Redis������Redisʵ�ֵ�ע�����ģ�ʹ�� Redis��Publish/Subscribe�¼�֪ͨ���ݱ����
Simple��Simpleע�����ı������һ����ͨ��Dubbo���񣬿��Լ��ٵ�����������ʹ����ͨѶ��ʽһ�£�
�����ص���ܹٷ��Ƽ���Zookeeperע�᷽ʽ�������Register����RegistryFactory�����ɵģ����忴һ�½ӿڶ��壻

##RegistryFactory�ӿ�
�ӿڶ������£�

```
@SPI("dubbo")
public interface RegistryFactory {
 
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
 
}
```
RegistryFactory�ṩ��SPI��չ��Ĭ��ʹ��dubbo����������Щ��չ���Բ鿴META-INF/dubbo/internal/com.alibaba.dubbo.registry.RegistryFactory��

```
dubbo=com.alibaba.dubbo.registry.dubbo.DubboRegistryFactory
multicast=com.alibaba.dubbo.registry.multicast.MulticastRegistryFactory
zookeeper=com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory
redis=com.alibaba.dubbo.registry.redis.RedisRegistryFactory
```
���Ƽ�ʹ�õ�ZookeeperΪʵ�����鿴ZookeeperRegistryFactory���ṩ��createRegistry������

```
private ZookeeperTransporter zookeeperTransporter;
 
public Registry createRegistry(URL url) {
       return new ZookeeperRegistry(url, zookeeperTransporter);
}
```
ʵ����ZookeeperRegistry�����������ֱ���url��zookeeperTransporter��zookeeperTransporter�ǲ���Zookeeper�Ŀͻ������������zkclient��curator���ַ�ʽ

```
@SPI("curator")
public interface ZookeeperTransporter {
 
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    ZookeeperClient connect(URL url);
 
}
```
ZookeeperTransporterͬ���ṩ��SPI��չ��Ĭ��ʹ��curator��ʽ���������ص㿴һ��Zookeeperע�����ġ�

##Zookeeperע������
###1.�����������
��dubbo����������У����Դ��²鿴Registry��Ĵ������̣�����ͨ��RegistryFactoryʵ����Registry��Registry���Խ���RegistryProtocol��������ע��(register)�Ͷ���(subscribe)��Ϣ��Ȼ��Registryͨ��ZKClient����Zookeeperָ����Ŀ¼��д��url��Ϣ������Ƕ�����ϢRegistry��ͨ��NotifyListener��֪ͨRegitryDirctory���и���url��������Cluster��ͨ��·�ɣ����ؾ���ѡ�������ṩ����

###2.Zookeeperע����������
�ٷ��ṩ��dubbo��Zookeeper���ĵ�����ͼ��
![ͼƬ����][2]

����˵����
�����ṩ������ʱ: ��/dubbo/com.foo.BarService/providersĿ¼��д���Լ���URL��ַ��
��������������ʱ: ����/dubbo/com.foo.BarService/providersĿ¼�µ��ṩ��URL��ַ������/dubbo/com.foo.BarService/consumersĿ¼��д���Լ���URL��ַ��
�����������ʱ: ����/dubbo/com.foo.BarService Ŀ¼�µ������ṩ�ߺ�������URL��ַ��
����ֱ��ע��(register)��ע��(unregister)������(subscribe)���˶�(unsubscribe)�ĸ�����������

###3.ע��(register)
ZookeeperRegistry�ĸ���FailbackRegistry��ʵ����register������FailbackRegistry�����ֿ��Կ��������У�ʧ���Զ��ָ�����̨��¼ʧ�����󣬶�ʱ�ط����ܣ�������忴һ��register������

```
public void register(URL url) {
        super.register(url);
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            // Sending a registration request to the server side
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;
 
            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
 
            // Record a failed registration request to a failed list, retry regularly
            failedRegistered.add(url);
        }
    }
```
��̨��¼��ʧ�ܵ����󣬰���failedRegistered��failedUnregistered��ע���ʱ�������ŵ�urlɾ����Ȼ��ִ��doRegister�������˷�ʽ��ZookeeperRegistry��ʵ�֣���Ҫ����Zookeeperָ����Ŀ¼��д��url��Ϣ�����ʧ�ܻ��¼ע��ʧ�ܵ�url���ȴ��Զ��ָ���doRegister��ش������£�

```
protected void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
}
```
����zkClient��create������Zookeeper�ϴ����ڵ㣬Ĭ�ϴ�����ʱ�ڵ㣬create������AbstractZookeeperClient��ʵ�֣�����Դ�����£�

```
public void create(String path, boolean ephemeral) {
       if (!ephemeral) {
           if (checkExists(path)) {
               return;
           }
       }
       int i = path.lastIndexOf('/');
       if (i > 0) {
           create(path.substring(0, i), false);
       }
       if (ephemeral) {
           createEphemeral(path);
       } else {
           createPersistent(path);
       }
   }
```
pathָ����Ҫ������Ŀ¼��ephemeralָ���Ƿ��Ǵ�����ʱ�ڵ㣬�����ṩ�˵ݹ鴴��Ŀ¼������Ҷ��Ŀ¼����Ŀ¼���ǳ־û��ģ����Է��ֲ����Ǵ�����ʱĿ¼���ǳ־û�Ŀ¼����û��ָ��Ŀ¼��Data������ʹ�õ���Ĭ��ֵ��Ҳ���Ǳ���ip��ַ��ʵ���д�����Ŀ¼���£�

```
/dubbo/com.dubboApi.DemoService/providers/dubbo%3A%2F%2F10.13.83.7%3A20880%2Fcom.dubboApi.DemoService%3Fanyhost%3Dtrue%26application%3Dhello-world-app%26dubbo%3D2.0.2%26generic%3Dfalse%26interface%3Dcom.dubboApi.DemoService%26methods%3DsyncSayHello%2CsayHello%2CasyncSayHello%26pid%3D13252%26serialization%3Dprotobuf%26side%3Dprovider%26timestamp%3D1545297239027
```
dubbo��һ�����ڵ㣬Ȼ����service���ƣ�providers�ǹ̶���һ�����ͣ���������Ѷ��������consumers��������һ����ʱ�ڵ㣻ʹ����ʱ�ڵ��Ŀ�ľ����ṩ�߳��ֶϵ���쳣ͣ��ʱ��ע���������Զ�ɾ���ṩ����Ϣ������ͨ�����·�����ѯ��ǰ��Ŀ¼�ڵ���Ϣ��

```
public class CuratorTest {
 
    static String path = "/dubbo";
    static CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
            .sessionTimeoutMs(5000).retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
 
    public static void main(String[] args) throws Exception {
        client.start();
        List<String> paths = listChildren(path);
        for (String path : paths) {
            Stat stat = new Stat();
            System.err.println(
                    "path:" + path + ",value:" + new String(client.getData().storingStatIn(stat).forPath(path)));
        }
    }
 
    private static List<String> listChildren(String path) throws Exception {
        List<String> pathList = new ArrayList<String>();
        pathList.add(path);
        List<String> list = client.getChildren().forPath(path);
        if (list != null && list.size() > 0) {
            for (String cPath : list) {
                String temp = "";
                if ("/".equals(path)) {
                    temp = path + cPath;
                } else {
                    temp = path + "/" + cPath;
                }
                pathList.addAll(listChildren(temp));
            }
        }
        return pathList;
    }
}
```
�ݹ����/dubboĿ¼�µ�������Ŀ¼��ͬʱ���ڵ�洢�����ݶ���ѯ������������£�

```
path:/dubbo,value:10.13.83.7
path:/dubbo/com.dubboApi.DemoService,value:10.13.83.7
path:/dubbo/com.dubboApi.DemoService/configurators,value:10.13.83.7
path:/dubbo/com.dubboApi.DemoService/providers,value:10.13.83.7
path:/dubbo/com.dubboApi.DemoService/providers/dubbo%3A%2F%2F10.13.83.7%3A20880%2Fcom.dubboApi.DemoService%3Fanyhost%3Dtrue%26application%3Dhello-world-app%26dubbo%3D2.0.2%26generic%3Dfalse%26interface%3Dcom.dubboApi.DemoService%26methods%3DsyncSayHello%2CsayHello%2CasyncSayHello%26pid%3D4712%26serialization%3Dprotobuf%26side%3Dprovider%26timestamp%3D1545358401966,value:10.13.83.7
```
�������һ���ڵ�����ʱ�ڵ㣬�������ǳ־û��ģ�

###4.ע��(unregister)
ͬ���ڸ���FailbackRegistry��ʵ����unregister�������������£�

```
public void unregister(URL url) {
       super.unregister(url);
       failedRegistered.remove(url);
       failedUnregistered.remove(url);
       try {
           // Sending a cancellation request to the server side
           doUnregister(url);
       } catch (Exception e) {
           Throwable t = e;
 
           // If the startup detection is opened, the Exception is thrown directly.
           boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                   && url.getParameter(Constants.CHECK_KEY, true)
                   && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
           boolean skipFailback = t instanceof SkipFailbackWrapperException;
           if (check || skipFailback) {
               if (skipFailback) {
                   t = t.getCause();
               }
               throw new IllegalStateException("Failed to unregister " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
           } else {
               logger.error("Failed to uregister " + url + ", waiting for retry, cause: " + t.getMessage(), t);
           }
 
           // Record a failed registration request to a failed list, retry regularly
           failedUnregistered.add(url);
       }
   }
```
ע��ʱͬ��ɾ����failedRegistered��failedUnregistered��ŵ�url��Ȼ�����doUnregister��ɾ��Zookeeper�е�Ŀ¼�ڵ㣬ʧ�ܵ�����»�洢��failedUnregistered�У��ȴ����ԣ�

```
protected void doUnregister(URL url) {
    try {
        zkClient.delete(toUrlPath(url));
    } catch (Throwable e) {
        throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
 
//CuratorZookeeperClientɾ������
public void delete(String path) {
    try {
        client.delete().forPath(path);
    } catch (NoNodeException e) {
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```
ֱ��ʹ��CuratorZookeeperClient�е�delete����ɾ����ʱ�ڵ㣻

###5.����(subscribe)
��������������ʱ��������Zookeeperע�������߽ڵ���Ϣ��Ȼ���ġ�/providersĿ¼���ṩ�ߵ�URL��ַ�����Ѷ�Ҳͬ����Ҫע��ڵ���Ϣ������Ϊ���������Ҫ�Է���˺����Ѷ˶����м�أ������ص㿴һ�¶��ĵ���ش��룬�ڸ���FailbackRegistry��ʵ����subscribe������

```
public void subscribe(URL url, NotifyListener listener) {
       super.subscribe(url, listener);
       removeFailedSubscribed(url, listener);
       try {
           // Sending a subscription request to the server side
           doSubscribe(url, listener);
       } catch (Exception e) {
           Throwable t = e;
 
           List<URL> urls = getCacheUrls(url);
           if (urls != null && !urls.isEmpty()) {
               notify(url, listener, urls);
               logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
           } else {
               // If the startup detection is opened, the Exception is thrown directly.
               boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                       && url.getParameter(Constants.CHECK_KEY, true);
               boolean skipFailback = t instanceof SkipFailbackWrapperException;
               if (check || skipFailback) {
                   if (skipFailback) {
                       t = t.getCause();
                   }
                   throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
               } else {
                   logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
               }
           }
 
           // Record a failed registration request to a failed list, retry regularly
           addFailedSubscribed(url, listener);
       }
   }
```
���Ƶĸ�ʽ��ͬ���洢��ʧ���˶���url��Ϣ���ص㿴ZookeeperRegistry�е�doSubscribe������

```
private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();
 
protected void doSubscribe(final URL url, final NotifyListener listener) {
       try {
           if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
               String root = toRootPath();
               ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
               if (listeners == null) {
                   zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                   listeners = zkListeners.get(url);
               }
               ChildListener zkListener = listeners.get(listener);
               if (zkListener == null) {
                   listeners.putIfAbsent(listener, new ChildListener() {
                       @Override
                       public void childChanged(String parentPath, List<String> currentChilds) {
                           for (String child : currentChilds) {
                               child = URL.decode(child);
                               if (!anyServices.contains(child)) {
                                   anyServices.add(child);
                                   subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                           Constants.CHECK_KEY, String.valueOf(false)), listener);
                               }
                           }
                       }
                   });
                   zkListener = listeners.get(listener);
               }
               zkClient.create(root, false);
               List<String> services = zkClient.addChildListener(root, zkListener);
               if (services != null && !services.isEmpty()) {
                   for (String service : services) {
                       service = URL.decode(service);
                       anyServices.add(service);
                       subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                               Constants.CHECK_KEY, String.valueOf(false)), listener);
                   }
               }
           } else {
               List<URL> urls = new ArrayList<URL>();
               for (String path : toCategoriesPath(url)) {
                   ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                   if (listeners == null) {
                       zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                       listeners = zkListeners.get(url);
                   }
                   ChildListener zkListener = listeners.get(listener);
                   if (zkListener == null) {
                       listeners.putIfAbsent(listener, new ChildListener() {
                           @Override
                           public void childChanged(String parentPath, List<String> currentChilds) {
                               ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                           }
                       });
                       zkListener = listeners.get(listener);
                   }
                   zkClient.create(path, false);
                   List<String> children = zkClient.addChildListener(path, zkListener);
                   if (children != null) {
                       urls.addAll(toUrlsWithEmpty(url, path, children));
                   }
               }
               notify(url, listener, urls);
           }
       } catch (Throwable e) {
           throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
       }
   }
```
��ZookeeperRegistry�ж�����һ��zkListeners������ÿ��URL��Ӧ��һ��map��map����ֱ���NotifyListener��ChildListener�Ķ�Ӧ��ϵ�����Ѷ˶���ʱ�����NotifyListener��ʵ����RegistryDirectory��ChildListener��һ���ڲ��࣬�����ڼ����Ľڵ㷢�����ʱ��֪ͨ��Ӧ�����Ѷˣ�����ļ�����������zkClient.addChildListener��ʵ�ֵģ�

```
public List<String> addChildListener(String path, final ChildListener listener) {
    ConcurrentMap<ChildListener, TargetChildListener> listeners = childListeners.get(path);
    if (listeners == null) {
        childListeners.putIfAbsent(path, new ConcurrentHashMap<ChildListener, TargetChildListener>());
        listeners = childListeners.get(path);
    }
    TargetChildListener targetListener = listeners.get(listener);
    if (targetListener == null) {
        listeners.putIfAbsent(listener, createTargetChildListener(path, listener));
        targetListener = listeners.get(listener);
    }
    return addTargetChildListener(path, targetListener);
}
 
public CuratorWatcher createTargetChildListener(String path, ChildListener listener) {
    return new CuratorWatcherImpl(listener);
}
 
public List<String> addTargetChildListener(String path, CuratorWatcher listener) {
    try {
        return client.getChildren().usingWatcher(listener).forPath(path);
    } catch (NoNodeException e) {
        return null;
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
 
private class CuratorWatcherImpl implements CuratorWatcher {
 
    private volatile ChildListener listener;
 
    public CuratorWatcherImpl(ChildListener listener) {
        this.listener = listener;
    }
 
    public void unwatch() {
        this.listener = null;
    }
 
    @Override
    public void process(WatchedEvent event) throws Exception {
        if (listener != null) {
            String path = event.getPath() == null ? "" : event.getPath();
            listener.childChanged(path,
                    StringUtils.isNotEmpty(path)
                            ? client.getChildren().usingWatcher(this).forPath(path)
                            : Collections.<String>emptyList());
        }
    }
}
```
CuratorWatcherImplʵ����Zookeeper�ļ����ӿ�CuratorWatcher�������ڽڵ��б��ʱ֪ͨ��Ӧ��ChildListener������ChildListener�Ϳ���֪ͨRegistryDirectory���и������ݣ�

###6.�˶�(unsubscribe)
�ڸ���FailbackRegistry��ʵ����unsubscribe����

```
public void unsubscribe(URL url, NotifyListener listener) {
        super.unsubscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // Sending a canceling subscription request to the server side
            doUnsubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;
 
            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true);
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to unsubscribe " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to unsubscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
 
            // Record a failed registration request to a failed list, retry regularly
            Set<NotifyListener> listeners = failedUnsubscribed.get(url);
            if (listeners == null) {
                failedUnsubscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
                listeners = failedUnsubscribed.get(url);
            }
            listeners.add(listener);
        }
    }
```
ͬ��ʹ��failedUnsubscribed�����洢ʧ���˶���url�����忴ZookeeperRegistry�е�doUnsubscribe����

```
protected void doUnsubscribe(URL url, NotifyListener listener) {
        ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
        if (listeners != null) {
            ChildListener zkListener = listeners.get(listener);
            if (zkListener != null) {
                if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                    String root = toRootPath();
                    zkClient.removeChildListener(root, zkListener);
                } else {
                    for (String path : toCategoriesPath(url)) {
                        zkClient.removeChildListener(path, zkListener);
                    }
                }
            }
        }
    }
```
�˶��ͱȽϼ��ˣ�ֻ��Ҫ�Ƴ��������Ϳ����ˣ�

###7.ʧ������
FailbackRegistry�����ֿ��Կ��������У�ʧ���Զ��ָ�����̨��¼ʧ�����󣬶�ʱ�ط����ܣ���FailbackRegistry�Ĺ�������������һ����ʱ����

```
this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
           @Override
           public void run() {
               // Check and connect to the registry
               try {
                   retry();
               } catch (Throwable t) { // Defensive fault tolerance
                   logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
               }
           }
       }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
```
ʵ������һ�����5��ִ��һ�����ԵĶ�ʱ����retry���ִ������£�

```
protected void retry() {
        if (!failedRegistered.isEmpty()) {
            Set<URL> failed = new HashSet<URL>(failedRegistered);
            if (failed.size() > 0) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry register " + failed);
                }
                try {
                    for (URL url : failed) {
                        try {
                            doRegister(url);
                            failedRegistered.remove(url);
                        } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                            logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                    logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
        ...ʡ��...
}
```
���ڼ���Ƿ����ʧ�ܵ�ע��(register)��ע��(unregister)������(subscribe)���˶�(unsubscribe)URL��������������ԣ�

##�ܽ�
�������Ƚ�����RegistryFactory, Registry, RegistryService�������Ľӿڣ�Ȼ����ZookeeperΪע�������ص������ע��(register)��ע��(unregister)������(subscribe)���˶�(unsubscribe)��ʽ��

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][3]
[https://gitee.com/OutOfMemory/blog][4]


  [1]: https://segmentfault.com/a/1190000017089603
  [2]: /img/bVbbUbN
  [3]: https://github.com/ksfzhaohui/blog
  [4]: https://gitee.com/OutOfMemory/blog