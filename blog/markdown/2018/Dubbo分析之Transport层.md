##ǰ��
��һƪ����[Dubbo����֮Serialize��][1]����������ײ�����л�/�����л��㣬���ļ�������Serialize�����һ��transport���紫��㣬�˲�ʹ�������е�һЩͨѶ��Դ���(ex:netty,mina,grizzly)�����ײ�ͨѶ������Ҳ���˼򵥽��ܣ����Ľ�����������˽⣻

##Transporter�����
dubboΪͨѶ����ṩ��ͳһ��bind��connet�ӿڣ�������й������չ����װ�ڽӿ��ࣺTransporter�У�

```
@SPI("netty")
public interface Transporter {
 
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;
 
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```
�ṩ��bind��connect�ӿڣ��ֱ��Ӧ��������˺Ϳͻ��ˣ���������Щʵ���࣬����ͼ��ʾ��

��Ĭ��ʹ�õ�netty���Ϊ�����������£�

```
public class NettyTransporter implements Transporter {
 
    public static final String NAME = "netty";
 
    @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
 
    @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }
 
}
```
����ķ������˷�װ��NettyServer�У��ͻ��˷�װ��NettyClient��url�����ǰ�����xml���õ���Ϣ(����������Ľӿڣ�ʹ�õ�Э�飬ʹ�õ����л���ʽ��ʹ�õ�ͨѶ��ܵ�)��listener��һ��Handler���ڽ���֮�����ݽ�������������ҵ������Ӧ���ϵļ���ͨѶ��Դ��ܣ��ֱ��ṩ�˶�Ӧ��Transporter������NettyTransporter��NettyTransporter(netty4)��MinaTransporter�Լ�GrizzlyTransporter������ʹ���������͵�Transporter����Transporters�����ṩ��getTransporter������

```
public static Transporter getTransporter() {
    return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
}
```
���ﲢû�����ڻ�ȡ����serialization��һ����ͨ����urlָ��transporter������Ȼ����ؾ����transporter�࣬����������һ����̬��transporter���ɴ˶�̬transporterȥ���ؾ�����ࣻ
��ΪServer�˺�Client���Էֱ����óɲ�ͬ��ͨѶ��ܣ�һ�λ�ȡΨһ��Transporter������������󣻾�������ɶ�̬����ķ�����ExtensionLoader��createAdaptiveExtensionClassCode�����У��˴������г�Դ�룬�ڴ�չʾһ��Ĭ�����ɵĶ�̬������չ�ࣺ

```
package com.alibaba.dubbo.remoting;
 
import com.alibaba.dubbo.common.extension.ExtensionLoader;
 
public class Transporter$Adaptive implements com.alibaba.dubbo.remoting.Transporter {
    public com.alibaba.dubbo.remoting.Server bind(
        com.alibaba.dubbo.common.URL arg0,
        com.alibaba.dubbo.remoting.ChannelHandler arg1)
        throws com.alibaba.dubbo.remoting.RemotingException {
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
 
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("server",
                url.getParameter("transporter", "netty"));
 
        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" +
                url.toString() + ") use keys([server, transporter])");
        }
 
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class)
                                                                                                                   .getExtension(extName);
 
        return extension.bind(arg0, arg1);
    }
 
    public com.alibaba.dubbo.remoting.Client connect(
        com.alibaba.dubbo.common.URL arg0,
        com.alibaba.dubbo.remoting.ChannelHandler arg1)
        throws com.alibaba.dubbo.remoting.RemotingException {
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
 
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("client",
                url.getParameter("transporter", "netty"));
 
        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" +
                url.toString() + ") use keys([client, transporter])");
        }
 
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class)
                                                                                                                   .getExtension(extName);
 
        return extension.connect(arg0, arg1);
    }
}
```
���Է���Server�˿���ͨ��transporter��server����������������չ�࣬����server�������õ�ֵ�ǿ��Ը���transporter������ֵ��ͬ��ClientҲ���ƣ���󲻹���bind()����connet()����ͨ��ExtensionLoader��getExtension��������ȡ�����transporter�ࣻͬserialize�㣬��ص�transporterҲͬ��������META-INF/dubbo/internal/com.alibaba.dubbo.remoting.Transporter�ļ��У�

```
netty=com.alibaba.dubbo.remoting.transport.netty.NettyTransporter
netty4=com.alibaba.dubbo.remoting.transport.netty4.NettyTransporter
mina=com.alibaba.dubbo.remoting.transport.mina.MinaTransporter
grizzly=com.alibaba.dubbo.remoting.transport.grizzly.GrizzlyTransporter
```

##Server�˺�Client����
###1.Server��
��ʵ���������Server��ʱ�������ȵ��ø���Ĺ����������в�����ʼ����ͬʱ����bind()����������������������AbstractServer���������£�

```
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);
        localAddress = getUrl().toInetSocketAddress();
 
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
            bindIp = NetUtils.ANYHOST;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
        this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
        this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
        try {
            doOpen();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
            }
        } catch (Throwable t) {
            throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
        }
        //fixme replace this with better method
        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
    }
```
��Ҫ��url��ȡ��������������ip��port��accepts(�ɽ��ܵ���������0��ʾ��������������Ĭ��Ϊ0)��idleTimeout�ȣ�Ȼ�����doOpen����ͨ�������ͨѶ��ܰ󶨶˿�����������Ĭ��ʹ�õ�NettyΪ�����鿴doOpen()�������£�

```
protected void doOpen() throws Throwable {
       NettyHelper.setNettyLoggerFactory();
       ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
       ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
       ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
       bootstrap = new ServerBootstrap(channelFactory);
 
       final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
       channels = nettyHandler.getChannels();
       // https://issues.jboss.org/browse/NETTY-365
       // https://issues.jboss.org/browse/NETTY-379
       // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
       bootstrap.setOption("child.tcpNoDelay", true);
       bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
           @Override
           public ChannelPipeline getPipeline() {
               NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
               ChannelPipeline pipeline = Channels.pipeline();
               /*int idleTimeout = getIdleTimeout();
               if (idleTimeout > 10000) {
                   pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
               }*/
               pipeline.addLast("decoder", adapter.getDecoder());
               pipeline.addLast("encoder", adapter.getEncoder());
               pipeline.addLast("handler", nettyHandler);
               return pipeline;
           }
       });
       // bind
       channel = bootstrap.bind(getBindAddress());
   }
```
�����ǳ��������netty������Ҫָ�����������nettyHandler��������Ѿ��������н��ܹ��ˣ��˴�������ϸ���ܣ��ص����nettyHandler��server�������ݾ�������֮��ͽ���NettyHandler������NettyHandler�̳���Netty��SimpleChannelHandler�࣬��д��channelConnected��channelDisconnected��messageReceived��writeRequested�Լ�exceptionCaught�����������Ͼ��ǳ���ļ��ֲ������������ӣ��Ͽ����ӣ�������Ϣ��������Ϣ���쳣������һ�²���Դ�룺

```
@Override
   public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           if (channel != null) {
               channels.put(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()), channel);
           }
           handler.connected(channel);
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
 
   @Override
   public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           channels.remove(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()));
           handler.disconnected(channel);
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
    @Override
   public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           handler.received(channel, e.getMessage());
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
 
   @Override
   public void writeRequested(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
       super.writeRequested(ctx, e);
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           handler.sent(channel, e.getMessage());
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
 
   @Override
   public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           handler.caught(channel, e.getCause());
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
```
��nettyԭ����channel��װ��dubbo��NettyChannel��ͬʱ��NettyChannel������NettyChannel���ڲ���̬����channelMap�У�����ķ�����ͳһ������getOrAddChannel����������ӽ�ȥ�������finally���ж�channel�Ƿ��Ѿ��رգ�����رմ�channelMap���Ƴ����м䲿�ֵ�����handler��Ӧ�ķ������˴���handler������ʵ����ʱ�����NettyServer��NettyServer����Ҳ��һ��ChannelHandler�����Կ�һ��channelHandler�ӿ��ࣺ

```
public interface ChannelHandler {
 
    void connected(Channel channel) throws RemotingException;
 
    void disconnected(Channel channel) throws RemotingException;
 
    void sent(Channel channel, Object message) throws RemotingException;
 
    void received(Channel channel, Object message) throws RemotingException;
 
    void caught(Channel channel, Throwable exception) throws RemotingException;
}
```
�����server����Ҳ������һЩ��������connectedʱ�ж��Ƿ񳬹�accepts����������ܾ����ӣ�������֮�󽻸�ʵ����Serverʱ�����ChannelHandler���������������HeaderExchanger�б���ʼ���ģ�

```
public class HeaderExchanger implements Exchanger {
 
    public static final String NAME = "header";
 
    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
 
    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
 
}
```
���Է�����������ChannelHandler��DecodeHandler��ע�����Decode��Netty�����decode��һ����Netty�����decode��ִ��NettyHandler֮ǰ��ִ�н����ˣ������Ĳ�����Exchange����д���������ʱ�Ȳ������ܣ�

###2.Client��
ͬ���鿴����AbstractClient�����췽�����£�

```
public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);
 
        send_reconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);
 
        shutdown_timeout = url.getParameter(Constants.SHUTDOWN_TIMEOUT_KEY, Constants.DEFAULT_SHUTDOWN_TIMEOUT);
 
        // The default reconnection interval is 2s, 1800 means warning interval is 1 hour.
        reconnect_warning_period = url.getParameter("reconnect.waring.period", 1800);
 
        try {
            doOpen();
        } catch (Throwable t) {
            close();
            throw new RemotingException(url.toInetSocketAddress(), null,
                    "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                            + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
        }
        try {
            // connect.
            connect();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress() + " connect to the server " + getRemoteAddress());
            }
        } catch (RemotingException t) {
            if (url.getParameter(Constants.CHECK_KEY, true)) {
                close();
                throw t;
            } else {
                logger.warn("Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                        + " connect to the server " + getRemoteAddress() + " (check == false, ignore and retry later!), cause: " + t.getMessage(), t);
            }
        } catch (Throwable t) {
            close();
            throw new RemotingException(url.toInetSocketAddress(), null,
                    "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                            + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
        }
 
        executor = (ExecutorService) ExtensionLoader.getExtensionLoader(DataStore.class)
                .getDefaultExtension().get(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
        ExtensionLoader.getExtensionLoader(DataStore.class)
                .getDefaultExtension().remove(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
    }
```
�ͻ�����Ҫ�ṩ�������ƣ����Գ�ʼ���ļ����������������йأ�send_reconnect��ʾ�ڷ�����Ϣʱ���������Ѿ��Ͽ��Ƿ���������reconnect_warning_period��ʾ��ñ�һ���������棬shutdown_timeout��ʾ���ӷ�����һֱ���Ӳ��ϵĳ�ʱʱ�䣻���������ǵ���doOpen()������ͬ����NettyΪ����

```
protected void doOpen() throws Throwable {
       NettyHelper.setNettyLoggerFactory();
       bootstrap = new ClientBootstrap(channelFactory);
       // config
       // @see org.jboss.netty.channel.socket.SocketChannelConfig
       bootstrap.setOption("keepAlive", true);
       bootstrap.setOption("tcpNoDelay", true);
       bootstrap.setOption("connectTimeoutMillis", getTimeout());
       final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
       bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
           @Override
           public ChannelPipeline getPipeline() {
               NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
               ChannelPipeline pipeline = Channels.pipeline();
               pipeline.addLast("decoder", adapter.getDecoder());
               pipeline.addLast("encoder", adapter.getEncoder());
               pipeline.addLast("handler", nettyHandler);
               return pipeline;
           }
       });
   }
```
Netty�ͻ��˵ĳ�����룬�����˺�Server����ͬ��NettyHandler��decoder��encoder�������ص㿴��connect������

```
protected void connect() throws RemotingException {
        connectLock.lock();
        try {
            if (isConnected()) {
                return;
            }
            initConnectStatusCheckCommand();
            doConnect();
            if (!isConnected()) {
                throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                        + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                        + ", cause: Connect wait timeout: " + getTimeout() + "ms.");
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info("Successed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                            + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                            + ", channel is " + this.getChannel());
                }
            }
            reconnect_count.set(0);
            reconnect_error_log_flag.set(false);
        } catch (RemotingException e) {
            throw e;
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                    + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                    + ", cause: " + e.getMessage(), e);
        } finally {
            connectLock.unlock();
        }
    }
```
�����ж��Ƿ��Ѿ����ӣ��������ֱ��return����������ʼ������״̬����������ڼ��channel�Ƿ����ӣ����ӶϿ��������������������������£�


```
private synchronized void initConnectStatusCheckCommand() {
        //reconnect=false to close reconnect
        int reconnect = getReconnectParam(getUrl());
        if (reconnect > 0 && (reconnectExecutorFuture == null || reconnectExecutorFuture.isCancelled())) {
            Runnable connectStatusCheckCommand = new Runnable() {
                @Override
                public void run() {
                    try {
                        if (!isConnected()) {
                            connect();
                        } else {
                            lastConnectedTime = System.currentTimeMillis();
                        }
                    } catch (Throwable t) {
                        String errorMsg = "client reconnect to " + getUrl().getAddress() + " find error . url: " + getUrl();
                        // wait registry sync provider list
                        if (System.currentTimeMillis() - lastConnectedTime > shutdown_timeout) {
                            if (!reconnect_error_log_flag.get()) {
                                reconnect_error_log_flag.set(true);
                                logger.error(errorMsg, t);
                                return;
                            }
                        }
                        if (reconnect_count.getAndIncrement() % reconnect_warning_period == 0) {
                            logger.warn(errorMsg, t);
                        }
                    }
                }
            };
            reconnectExecutorFuture = reconnectExecutorService.scheduleWithFixedDelay(connectStatusCheckCommand, reconnect, reconnect, TimeUnit.MILLISECONDS);
        }
    }
```
������һ��Runnable����������Ƿ����ӣ�������ӶϿ�������connect��������ʱ���Ƚ���ScheduledThreadPoolExecutor��ִ�У���ʼ��֮��͵��þ���Client��doConnect������Ҳ��ͨѶ��ܵ�һЩ������룬�˴����г��ˣ���������NettyChannel�Ľ��ܺ�Server�����ƣ���������н��ܣ�

##�ܽ�
�����ص������dubbo�ܹ��е�transport�㣬����Χ��Transporter, Client, Server��ChannelHandler������չ�������ں����Ĵ�����exchange��Ϣ�����㣻

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][2]
[https://gitee.com/OutOfMemory/blog][3]


  [1]: https://segmentfault.com/a/1190000016620859
  [2]: https://github.com/ksfzhaohui/blog
  [3]: https://gitee.com/OutOfMemory/blog