##ϵ������
[Dubbo����Serialize��][1]
[Dubbo����֮Transport��][2]
[Dubbo����֮Exchange ��][3]

##ǰ��
����������Dubbo����֮Transport�㣬���ļ�������Exchange�㣬�˲�ٷ�����Ϊ��Ϣ�����㣺��װ������Ӧģʽ��ͬ��ת�첽���� Request, Response Ϊ���ģ���չ�ӿ�Ϊ Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer������ֱ���н���

##Exchanger����
Exchanger�Ǵ˲�ĺ��Ľӿ��࣬�ṩ��connect()��bind()�ӿڣ��ֱ𷵻�ExchangeClient��ExchangeServer��dubbo�ṩ�˴˽ӿڵ�Ĭ��ʵ����HeaderExchanger���������£�

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
��ʵ��������connect��bind�зֱ�ʵ������HeaderExchangeClient��HeaderExchangeServer������Ĳ�����Transporters��������Ϊ�������Transport�������ࣻ�����ExchangeClient/ExchangeServer��ʵ���Ƕ�Client/Server�İ�װ��ͬʱ�������Լ���ChannelHandler��ChannelHandler�Ѿ���Transport����ܹ��ˣ��ṩ�����ӽ��������Ӷ˿ڣ��������󣬽�������Ƚӿڣ���Ĭ��ʹ�õ�NettyΪ����������Ƕ�NettyClient��NettyServer�İ�װ��ͬʱ����DecodeHandler����NettyHandler�б����ã�

##ExchangeClient����
ExchangeClient����Ҳ�̳���Client��ͬʱҲ�̳���ExchangeChannel��

```
public interface ExchangeClient extends Client, ExchangeChannel {
 
}
 
public interface ExchangeChannel extends Channel {
 
    ResponseFuture request(Object request) throws RemotingException;
 
    ResponseFuture request(Object request, int timeout) throws RemotingException;
 
    ExchangeHandler getExchangeHandler();
 
    @Override
    void close(int timeout);
 
}
```
ExchangeChannel�����ϲ��data��װ��Request��Ȼ���͸�Transport�㣻������߼���HeaderExchangeChannel�У�

```
public ResponseFuture request(Object request, int timeout) throws RemotingException {
       if (closed) {
           throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
       }
       // create request.
       Request req = new Request();
       req.setVersion(Version.getProtocolVersion());
       req.setTwoWay(true);
       req.setData(request);
       DefaultFuture future = new DefaultFuture(channel, req, timeout);
       try {
           channel.send(req);
       } catch (RemotingException e) {
           future.cancel();
           throw e;
       }
       return future;
   }
```
������һ��Request���ڹ�������ͬʱ�����һ��RequestId��������Э��汾���Ƿ�˫��ͨ�ţ������������ʵ��ҵ�����ݣ�������ʵ������һ��DefaultFuture�࣬����ʵ����ͬ��ת�첽�ķ�ʽ��channel����send��������֮�󣬲���Ҫ�ȴ������ֱ�ӽ�DefaultFuture���ظ��ϲ㣬�ϲ����ͨ������DefaultFuture��get��������ȡ��Ӧ��get�����������ȴ���ȡ����������Ӧ�Ż᷵�أ�Client������Ϣ��handler���棬����Netty��NettyHandler����messageReceived����������Ӧ��Ϣ��NettyHandler���ջ�������洫���DecodeHandler��DecodeHandler�����ж�һ���Ƿ��Ѿ����룬��������ֱ�ӵ���HeaderExchangeHandler��Ĭ���Ѿ������˱�������������Ի�ֱ�ӵ���HeaderExchangeHandler�����received������

```
public void received(Channel channel, Object message) throws RemotingException {
       channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
       ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
       try {
           if (message instanceof Request) {
               // handle request.
               Request request = (Request) message;
               if (request.isEvent()) {
                   handlerEvent(channel, request);
               } else {
                   if (request.isTwoWay()) {
                       Response response = handleRequest(exchangeChannel, request);
                       channel.send(response);
                   } else {
                       handler.received(exchangeChannel, request.getData());
                   }
               }
           } else if (message instanceof Response) {
               handleResponse(channel, (Response) message);
           } else if (message instanceof String) {
               if (isClientSide(channel)) {
                   Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                   logger.error(e.getMessage(), e);
               } else {
                   String echo = handler.telnet(channel, (String) message);
                   if (echo != null && echo.length() > 0) {
                       channel.send(echo);
                   }
               }
           } else {
               handler.received(exchangeChannel, message);
           }
       } finally {
           HeaderExchangeChannel.removeChannelIfDisconnected(channel);
       }
   }
```
����˺Ϳͻ��˶���ʹ�ô˷����������ǿͻ��˽��ܵ���Response��ֱ�ӵ���handleResponse������

```
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}
```
���յ���Ӧ֮����ȥ����DefaultFuture�Ѿ��յ���Ӧ��DefaultFuture��������requestId��ӦDefaultFuture��һ��ConcurrentHashMap��������ôӳ���ȥ��ResponseҲ����һ��responseId����responseId��requestId����ͬ�ģ�

```
private final Lock lock = new ReentrantLock();
private final Condition done = lock.newCondition();
   
public static void received(Channel channel, Response response) {
      try {
          DefaultFuture future = FUTURES.remove(response.getId());
          if (future != null) {
              future.doReceived(response);
          } else {
              logger.warn("The timeout response finally returned at "
                      + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                      + ", response " + response
                      + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                      + " -> " + channel.getRemoteAddress()));
          }
      } finally {
          CHANNELS.remove(response.getId());
      }
  }
   
  private void doReceived(Response res) {
      lock.lock();
      try {
          response = res;
          if (done != null) {
              done.signal();
          }
      } finally {
          lock.unlock();
      }
      if (callback != null) {
          invokeCallback(callback);
      }
  }
```
ͨ��responseId��ȡ��֮ǰ����ʱ������DefaultFuture��Ȼ���ٸ���DefaultFuture�ڲ���response���󣬸�����֮���ڵ���Condition��signal�������û�����ͨ��DefaultFuture��get������ȡ��Ӧ�������̣߳�

```
public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        return returnFromResponse();
    }
```
���Է�������Ҫô����ȡ��signal�������ѣ�Ҫô�ȴ���ʱ�����ϴ����ǿͻ��˷��ͻ�ȡ��Ӧ�����̣����濴��������������

##ExchangeServer����
ExchangeServer�̳���Server��ͬʱ�ṩ��������װ�����Channel�ķ���

```
public interface ExchangeServer extends Server {
 
    Collection<ExchangeChannel> getExchangeChannels();
 
    ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);
}
```
����������Ҫ���ڽ���Request��Ϣ��Ȼ������Ϣ��������Ӧ���͸��ͻ��ˣ���ؽ�����Ϣ�Ѿ���������ܹ��ˣ�ͬ������HeaderExchangeHandler�����received�����У�ֻ�����������Ϣ����ΪRequest��

```
Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
      Response res = new Response(req.getId(), req.getVersion());
      if (req.isBroken()) {
          Object data = req.getData();
 
          String msg;
          if (data == null) msg = null;
          else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
          else msg = data.toString();
          res.setErrorMessage("Fail to decode request due to: " + msg);
          res.setStatus(Response.BAD_REQUEST);
 
          return res;
      }
      // find handler by message class.
      Object msg = req.getData();
      try {
          // handle data.
          Object result = handler.reply(channel, msg);
          res.setStatus(Response.OK);
          res.setResult(result);
      } catch (Throwable e) {
          res.setStatus(Response.SERVICE_ERROR);
          res.setErrorMessage(StringUtils.toString(e));
      }
      return res;
  }
```
���ȴ�����һ��Response������ָ��responseIdΪrequestId�������ڿͻ��˶�λ�������DefaultFuture��Ȼ�����handler��reply����������Ϣ�����ؽ������δ���Ľ��ں����protocol����ܣ����¾���ͨ��Request����Ϣ���������Server�˵ķ���Ȼ�󷵻ؽ����Ȼ�󽫽������Response�����У�ͨ��channel����Ϣ���Ϳͻ��ˣ�

##�ܽ�
���Ľ�����Exchange��Ĵ������̣�Χ��Exchanger��ExchangeClient��ExchangeServerչ���������װ��Request����Ӧ��װ��Response���ͻ���ͨ���첽�ķ�ʽ���շ���������

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][4]
[https://gitee.com/OutOfMemory/blog][5]


  [1]: https://segmentfault.com/a/1190000016620859
  [2]: https://segmentfault.com/a/1190000016777060
  [3]: https://segmentfault.com/a/1190000016802475
  [4]: https://github.com/ksfzhaohui/blog
  [5]: https://gitee.com/OutOfMemory/blog