##ϵ������
[Dubbo����Serialize��][1]
[Dubbo����֮Transport��][2]
[Dubbo����֮Exchange��][3]
[Dubbo����֮Protocol��][4]

##ǰ��
��������[Dubbo����֮Protocol��][5]�����ļ�������dubbo��cluster�㣬�˲��װ����ṩ�ߵ�·�ɼ����ؾ��⣬���Ž�ע�����ģ���InvokerΪ���ģ���չ�ӿ�ΪCluster, Directory, Router, LoadBalance��

##Cluster�ӿ�
����cluster�����ʹ������ͼƬ������
![ͼƬ����][6]

���ڵ��ϵ��
�����Invoker��Provider��һ���ɵ���Service�ĳ���Invoker��װ��Provider��ַ��Service�ӿ���Ϣ��
Directory������Invoker�����԰�������List ������List��ͬ���ǣ�����ֵ�����Ƕ�̬�仯�ģ�����ע���������ͱ����
Cluster��Directory�еĶ��Invokerαװ��һ�� Invoker�����ϲ�͸����αװ���̰������ݴ��߼�������ʧ�ܺ�������һ����
Router����Ӷ��Invoker�а�·�ɹ���ѡ���Ӽ��������д���룬Ӧ�ø���ȣ�
LoadBalance����Ӷ��Invoker��ѡ�������һ�����ڱ��ε��ã�ѡ�Ĺ��̰����˸��ؾ����㷨������ʧ�ܺ���Ҫ��ѡ��

Cluster����Ŀ¼��·�ɣ����ؾ����ȡ��һ�����õ�Invoker�������ϲ���ã��ӿ����£�

```
@SPI(FailoverCluster.NAME)
public interface Cluster {
 
    /**
     * Merge the directory invokers to a virtual invoker.
     *
     * @param <T>
     * @param directory
     * @return cluster invoker
     * @throws RpcException
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
 
}
```
Cluster��һ����Ⱥ�ݴ�ӿڣ�����·�ɣ����ؾ���֮���ȡ��Invoker�����ݴ����������dubbo�ṩ�˶����ݴ���ư�����
**Failover Cluster��**ʧ���Զ��л���������ʧ�ܣ��������������� [1]��ͨ�����ڶ������������Ի���������ӳ١���ͨ�� retries=��2�� ���������Դ���(������һ��)��
**Failfast Cluster��**����ʧ�ܣ�ֻ����һ�ε��ã�ʧ����������ͨ�����ڷ��ݵ��Ե�д����������������¼��
**Failsafe Cluster��**ʧ�ܰ�ȫ�������쳣ʱ��ֱ�Ӻ��ԡ�ͨ������д�������־�Ȳ�����
**Failback Cluster��**ʧ���Զ��ָ�����̨��¼ʧ�����󣬶�ʱ�ط���ͨ��������Ϣ֪ͨ������
**Forking Cluster��**���е��ö����������ֻҪһ���ɹ������ء�ͨ������ʵʱ��Ҫ��ϸߵĶ�����������Ҫ�˷Ѹ��������Դ����ͨ�� forks=��2�� ���������������
**Broadcast Cluster��**�㲥���������ṩ�ߣ�������ã�����һ̨�����򱨴� [2]��ͨ������֪ͨ�����ṩ�߸��»������־�ȱ�����Դ��Ϣ��

Ĭ��ʹ����FailoverCluster��ʧ�ܵ�ʱ���Ĭ������������������Ĭ��Ϊ���Σ���ȻҲ������չ�������ݴ���ƣ���һ��Ĭ�ϵ�FailoverCluster�ݴ���ƣ�����Դ����FailoverClusterInvoker�У�

```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
       List<Invoker<T>> copyinvokers = invokers;
       checkInvokers(copyinvokers, invocation);
       int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
       if (len <= 0) {
           len = 1;
       }
       // retry loop.
       RpcException le = null; // last exception.
       List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
       Set<String> providers = new HashSet<String>(len);
       for (int i = 0; i < len; i++) {
           //Reselect before retry to avoid a change of candidate `invokers`.
           //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
           if (i > 0) {
               checkWhetherDestroyed();
               copyinvokers = list(invocation);
               // check again
               checkInvokers(copyinvokers, invocation);
           }
           Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
           invoked.add(invoker);
           RpcContext.getContext().setInvokers((List) invoked);
           try {
               Result result = invoker.invoke(invocation);
               if (le != null && logger.isWarnEnabled()) {
                   logger.warn("Although retry the method " + invocation.getMethodName()
                           + " in the service " + getInterface().getName()
                           + " was successful by the provider " + invoker.getUrl().getAddress()
                           + ", but there have been failed providers " + providers
                           + " (" + providers.size() + "/" + copyinvokers.size()
                           + ") from the registry " + directory.getUrl().getAddress()
                           + " on the consumer " + NetUtils.getLocalHost()
                           + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                           + le.getMessage(), le);
               }
               return result;
           } catch (RpcException e) {
               if (e.isBiz()) { // biz exception.
                   throw e;
               }
               le = e;
           } catch (Throwable e) {
               le = new RpcException(e.getMessage(), e);
           } finally {
               providers.add(invoker.getUrl().getAddress());
           }
       }
       throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
               + invocation.getMethodName() + " in the service " + getInterface().getName()
               + ". Tried " + len + " times of the providers " + providers
               + " (" + providers.size() + "/" + copyinvokers.size()
               + ") from the registry " + directory.getUrl().getAddress()
               + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
               + Version.getVersion() + ". Last error is: "
               + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
   }
```
invocation�ǿͻ��˴�������������ز�������(�������ƣ���������������ֵ��������Ϣ)��invokers�Ǿ���·��֮��ķ������б�loadbalance��ָ���ĸ��ؾ�����ԣ����ȼ��invokers�Ƿ�Ϊ�գ�Ϊ��ֱ�����쳣��Ȼ���ȡ���ԵĴ���Ĭ��Ϊ2�Σ�����������ѭ������ָ��������������ǵ�һ�ε���(��ʾ��һ�ε���ʧ��)�������¼��ط������б�Ȼ��ͨ�����ؾ�����Ի�ȡΨһ��Invoker��������ͨ��Invoker��invocation���͸������������ؽ��Result��

�����doInvoke�������ڳ�����AbstractClusterInvoker�б����õģ�

```
public Result invoke(final Invocation invocation) throws RpcException {
       checkWhetherDestroyed();
       LoadBalance loadbalance = null;
       List<Invoker<T>> invokers = list(invocation);
       if (invokers != null && !invokers.isEmpty()) {
           loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                   .getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
       }
       RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
       return doInvoke(invocation, invokers, loadbalance);
   }
    
    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
       List<Invoker<T>> invokers = directory.list(invocation);
       return invokers;
   }
```
����ͨ��Directory��ȡInvoker�б�ͬʱ��Directory��Ҳ����·�ɴ���Ȼ���ȡ���ؾ�����ԣ������þ�����ݴ���ԣ�������忴һ��Directory��

##Directory�ӿ�
�ӿڶ������£�

```
public interface Directory<T> extends Node {
 
    /**
     * get service type.
     *
     * @return service type.
     */
    Class<T> getInterface();
 
    /**
     * list invokers.
     *
     * @return invokers
     */
    List<Invoker<T>> list(Invocation invocation) throws RpcException;
 
}
```
Ŀ¼�������þ��ǻ�ȡָ���ӿڵķ����б�����ʵ����������StaticDirectory��RegistryDirectory��ͬʱ���̳���AbstractDirectory�������ֿ��Դ���֪��StaticDirectory��һ���̶���Ŀ¼���񣬱�ʾ�����Invoker�б��ᶯ̬�ı䣻RegistryDirectory��һ����̬��Ŀ¼����ͨ��ע�����Ķ�̬���·����б�listʵ���ڳ������У�

```
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
       if (destroyed) {
           throw new RpcException("Directory already destroyed .url: " + getUrl());
       }
       List<Invoker<T>> invokers = doList(invocation);
       List<Router> localRouters = this.routers; // local reference
       if (localRouters != null && !localRouters.isEmpty()) {
           for (Router router : localRouters) {
               try {
                   if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                       invokers = router.route(invokers, getConsumerUrl(), invocation);
                   }
               } catch (Throwable t) {
                   logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
               }
           }
       }
       return invokers;
   }
```
���ȼ��Ŀ¼�Ƿ����٣�Ȼ�����doList��������ʵ�����ж��壬������·�ɹ��ܣ������ص㿴һ��StaticDirectory��RegistryDirectory�е�doList����

###1.RegistryDirectory
��һ����̬��Ŀ¼�������п��Կ���RegistryDirectoryͬʱҲ�̳���NotifyListener�ӿڣ���һ��֪ͨ�ӿڣ�ע�������з����б���µ�ʱ��ͬʱ֪ͨRegistryDirectory��֪ͨ�߼����£�

```
public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category)
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators
        if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
        // routers
        if (routerUrls != null && !routerUrls.isEmpty()) {
            List<Router> routers = toRouters(routerUrls);
            if (routers != null) { // null - do nothing
                setRouters(routers);
            }
        }
        List<Configurator> localConfigurators = this.configurators; // local reference
        // merge override parameters
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && !localConfigurators.isEmpty()) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
        // providers
        refreshInvoker(invokerUrls);
    }
```
��֪ͨ�ӿڻ������������url������router(·��)��configurator(����)��provider(�����ṩ��)��
·�ɹ��򣺾���һ��dubbo������õ�Ŀ�����������Ϊ����·�ɹ���ͽű�·�ɹ��򣬲���֧�ֿ���չ����ע������д��·�ɹ���Ĳ���ͨ���ɼ�����Ļ��������ĵ�ҳ����ɣ�
���ù�����ע������д�붯̬���ø��ǹ��� [1]���ù���ͨ���ɼ�����Ļ��������ĵ�ҳ����ɣ�
provider����̬�ṩ�ķ����б�
·�ɹ�������ù�����ʵ���Ƕ�provider�����б���º͹��˴���refreshInvoker�������Ǹ�������url���ˢ�±��ص�invoker�б����濴һ��RegistryDirectoryʵ�ֵ�doList�ӿڣ�

```
public List<Invoker<T>> doList(Invocation invocation) {
        if (forbidden) {
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                "No provider available from registry " + getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +  NetUtils.getLocalHost()
                        + " use dubbo version " + Version.getVersion() + ", please check status of providers(disabled, not registered or in blacklist).");
        }
        List<Invoker<T>> invokers = null;
        Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
        if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
            String methodName = RpcUtils.getMethodName(invocation);
            Object[] args = RpcUtils.getArguments(invocation);
            if (args != null && args.length > 0 && args[0] != null
                    && (args[0] instanceof String || args[0].getClass().isEnum())) {
                invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // The routing can be enumerated according to the first parameter
            }
            if (invokers == null) {
                invokers = localMethodInvokerMap.get(methodName);
            }
            if (invokers == null) {
                invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
            }
            if (invokers == null) {
                Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
                if (iterator.hasNext()) {
                    invokers = iterator.next();
                }
            }
        }
        return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
    }
```
refreshInvoker����֮�󣬷����б���methodInvokerMap���ڣ�һ��������Ӧ�����б�Map>>��
ͨ��Invocation��ָ���ķ�����ȡ��Ӧ�ķ����б��������ķ���û�ж�Ӧ�ķ����б����ȡ��*����Ӧ�ķ����б�������֮����ڸ����н���·�ɴ���·�ɹ���ͬ����ͨ��֪ͨ�ӿڻ�ȡ�ģ�·�ɹ��������½��ܣ�

###2.StaticDirectory
����һ����̬��Ŀ¼��������ķ����б��ڳ�ʼ����ʱ����Ѿ����ڣ����Ҳ���ı䣻StaticDirectory�õñȽ���,��Ҫ���ڷ���Զ�ע�����ĵ����ã�

```
protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
 
    return invokers;
}
```
��Ϊ�Ǿ�̬�ģ�����doList����Ҳ�ܼ򵥣�ֱ�ӷ����ڴ��еķ����б��ɣ�

##Router�ӿ�
·�ɹ������һ��dubbo������õ�Ŀ�����������Ϊ����·�ɹ���ͽű�·�ɹ��򣬲���֧�ֿ���չ���ӿ����£�

```
public interface Router extends Comparable<Router> {
 
    /**
     * get the router url.
     *
     * @return url
     */
    URL getUrl();
 
    /**
     * route.
     *
     * @param invokers
     * @param url        refer url
     * @param invocation
     * @return routed invokers
     * @throws RpcException
     */
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
 
}
```
�ӿ����ṩ��route����ͨ��һ���Ĺ�����˳�invokers��һ���Ӽ����ṩ������ʵ���ࣺScriptRouter��ConditionRouter��MockInvokersSelector
ScriptRouter���ű�·�ɹ���֧�� JDK �ű���������нű������磺javascript, jruby, groovy �ȣ�ͨ��type=javascript�������ýű����ͣ�ȱʡΪjavascript��
ConditionRouter�������������ʽ��·�ɹ����磺host = 10.20.153.10 => host = 10.20.153.11��=> ֮ǰ��Ϊ������ƥ�����������в����������ߵ� URL ���жԱȣ�=> ֮��Ϊ�ṩ�ߵ�ַ�б�Ĺ������������в������ṩ�ߵ� URL ���жԱȣ�
MockInvokersSelector���Ƿ�����Ϊʹ��mock����·������ֻ֤�о���Э��MOCK�ĵ����߳��������յĵ������б��У��������������߽����ų���

�����ص㿴һ��ScriptRouterԴ��

```
public ScriptRouter(URL url) {
       this.url = url;
       String type = url.getParameter(Constants.TYPE_KEY);
       this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
       String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
       if (type == null || type.length() == 0) {
           type = Constants.DEFAULT_SCRIPT_TYPE_KEY;
       }
       if (rule == null || rule.length() == 0) {
           throw new IllegalStateException(new IllegalStateException("route rule can not be empty. rule:" + rule));
       }
       ScriptEngine engine = engines.get(type);
       if (engine == null) {
           engine = new ScriptEngineManager().getEngineByName(type);
           if (engine == null) {
               throw new IllegalStateException(new IllegalStateException("Unsupported route rule type: " + type + ", rule: " + rule));
           }
           engines.put(type, engine);
       }
       this.engine = engine;
       this.rule = rule;
   }
```
�������ֱ��ʼ���ű�����(engine)�ͽű�����(rule)��Ĭ�ϵĽű�������javascript����һ�������url��
```
"script://0.0.0.0/com.foo.BarService?category=routers&dynamic=false&rule=" + URL.encode("��function route(invokers) { ... } (invokers)��")
```
scriptЭ���ʾһ���ű�Э�飬rule������һ��javascript�ű�������Ĳ�����invokers��

```
��function route(invokers) {
    var result = new java.util.ArrayList(invokers.size());
    for (i = 0; i < invokers.size(); i ++) {
        if ("10.20.153.10".equals(invokers.get(i).getUrl().getHost())) {
            result.add(invokers.get(i));
        }
    }
    return result;
} (invokers)��; // ��ʾ����ִ�з���
```
������νű����˳�hostΪ10.20.153.10�����������ִ����νű��ģ���route�����У�

```
public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
     try {
         List<Invoker<T>> invokersCopy = new ArrayList<Invoker<T>>(invokers);
         Compilable compilable = (Compilable) engine;
         Bindings bindings = engine.createBindings();
         bindings.put("invokers", invokersCopy);
         bindings.put("invocation", invocation);
         bindings.put("context", RpcContext.getContext());
         CompiledScript function = compilable.compile(rule);
         Object obj = function.eval(bindings);
         if (obj instanceof Invoker[]) {
             invokersCopy = Arrays.asList((Invoker<T>[]) obj);
         } else if (obj instanceof Object[]) {
             invokersCopy = new ArrayList<Invoker<T>>();
             for (Object inv : (Object[]) obj) {
                 invokersCopy.add((Invoker<T>) inv);
             }
         } else {
             invokersCopy = (List<Invoker<T>>) obj;
         }
         return invokersCopy;
     } catch (ScriptException e) {
         //fail then ignore rule .invokers.
         logger.error("route error , rule has been ignored. rule: " + rule + ", method:" + invocation.getMethodName() + ", url: " + RpcContext.getContext().getUrl(), e);
         return invokers;
     }
 }
```
����ͨ���ű��������ű���Ȼ��ִ�нű���ͬʱ����Bindings�����������ڽű��оͿ��Ի�ȡinvokers��Ȼ����й��ˣ��������һ�¸��ؾ������

##LoadBalance�ӿ�
�ڼ�Ⱥ���ؾ���ʱ��Dubbo�ṩ�˶��־�����ԣ�ȱʡΪrandom������ã�����������չ���ؾ�����ԣ��ӿ������£�

```
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
 
    /**
     * select one invoker in list.
     *
     * @param invokers   invokers.
     * @param url        refer url
     * @param invocation invocation.
     * @return selected invoker.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
 
}
```
SPI������Ĭ�ϵĲ���ΪRandomLoadBalance���ṩ��һ��select������ͨ�����Դӷ����б���ѡ��һ��invoker��dubboĬ���ṩ�˶��ֲ��ԣ�
**Random LoadBalance��**�������Ȩ������������ʣ���һ����������ײ�ĸ��ʸߣ���������Խ��ֲ�Խ���ȣ����Ұ�����ʹ��Ȩ�غ�Ҳ�ȽϾ��ȣ������ڶ�̬�����ṩ��Ȩ�أ�
**RoundRobin LoadBalance��**��ѯ������Լ���Ȩ��������ѯ���ʣ����������ṩ���ۻ���������⣬���磺�ڶ�̨������������û�ң�����������ڶ�̨ʱ�Ϳ����ǣ�
�ö���֮���������󶼿��ڵ����ڶ�̨�ϣ�
**LeastActive LoadBalance��**���ٻ�Ծ����������ͬ��Ծ�����������Ծ��ָ����ǰ������ʹ�����ṩ���յ�����������ΪԽ�����ṩ�ߵĵ���ǰ��������Խ��
**ConsistentHash LoadBalance**��һ���� Hash����ͬ�������������Ƿ���ͬһ�ṩ�ߣ���ĳһ̨�ṩ�߹�ʱ��ԭ���������ṩ�ߵ����󣬻�������ڵ㣬ƽ̯�������ṩ�ߣ�����������ұ䶯��

�����ص㿴һ��Ĭ�ϵ�RandomLoadBalanceԴ��

```
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // Number of invokers
        int totalWeight = 0; // The sum of weights
        boolean sameWeight = true; // Every invoker has the same weight?
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight; // Sum
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offset = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(random.nextInt(length));
    }
```
���ȼ�����Ȩ�أ�ͬʱ����Ƿ�ÿһ����������ͬ��Ȩ�أ������Ȩ�ش���0���ҷ����Ȩ�ض�����ͬ����ͨ��Ȩ�������ѡ�񣬷���ֱ��ͨ��Random�����������

##�ܽ�
����Χ��Cluster���еļ�����Ҫ�Ľӿڴ��ϵ������ֱ���ܣ����ص���������е�ĳЩʵ���ࣻ��Ϲٷ��ṩ�ĵ���ͼ�����Ǻ��������˲�ġ�


  [1]: https://segmentfault.com/a/1190000016620859
  [2]: https://segmentfault.com/a/1190000016777060
  [3]: https://segmentfault.com/a/1190000016802475
  [4]: https://segmentfault.com/a/1190000016885261
  [5]: https://segmentfault.com/a/1190000016885261
  [6]: /img/bV8Kih