**前言**  
最近想批量下载一些国外网站的视频，之前写过一个代理程序[shadowsocks-netty](https://gitee.com/OutOfMemory/shadowsocks-netty)，打算直接  
用它来当作客户端代理程序，而HttpClient4也支持Socks代理；所有准备用HttpClient4来访问国外网站和视频资源

HttpClient4版本

```
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.3.6</version>
</dependency>
```

**访问网站**  
设置代理ip和port分别是：localhost和1080  
访问国外网站hostname为：www.google.com  
具体代码如下：

```
public class ClientExecuteSOCKS {
 
    /** 代理参数 IP+PORT **/
    private static String PROXY_IP = "localhost";
    private static int PROXY_PORT = 1080;
 
    public static void main(String[] args) throws Exception {
        Registry<ConnectionSocketFactory> reg = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("http", new MyConnectionSocketFactory()).build();
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(reg);
        CloseableHttpClient httpclient = HttpClients.custom().setConnectionManager(cm).build();
        try {
            InetSocketAddress socksaddr = new InetSocketAddress(PROXY_IP, PROXY_PORT);
            HttpClientContext context = HttpClientContext.create();
            context.setAttribute("socks.address", socksaddr);
 
            HttpHost target = new HttpHost("www.google.com", 80, "http");
            HttpGet request = new HttpGet("/");
 
            System.out.println("Executing request " + request + " to " + target + " via SOCKS proxy " + socksaddr);
            CloseableHttpResponse response = httpclient.execute(target, request, context);
            try {
                System.out.println("----------------------------------------");
                System.out.println(response.getStatusLine());
                String htmlStr = EntityUtils.toString(response.getEntity());
                System.out.println(htmlStr);
            } finally {
                response.close();
            }
        } finally {
            httpclient.close();
        }
    }
 
    static class MyConnectionSocketFactory implements ConnectionSocketFactory {
 
        public Socket createSocket(final HttpContext context) throws IOException {
            InetSocketAddress socksaddr = (InetSocketAddress) context.getAttribute("socks.address");
            Proxy proxy = new Proxy(Proxy.Type.SOCKS, socksaddr);
            return new Socket(proxy);
        }
 
        public Socket connectSocket(final int connectTimeout, final Socket socket, final HttpHost host,
                final InetSocketAddress remoteAddress, final InetSocketAddress localAddress, final HttpContext context)
                throws IOException, ConnectTimeoutException {
            Socket sock;
            if (socket != null) {
                sock = socket;
            } else {
                sock = createSocket(context);
            }
            if (localAddress != null) {
                sock.bind(localAddress);
            }
            try {
                sock.connect(remoteAddress, connectTimeout);
            } catch (SocketTimeoutException ex) {
                throw new ConnectTimeoutException(ex, host, remoteAddress.getAddress());
            }
            return sock;
        }
    }
}
```

以上代码是Httpclient提供的实例，稍作修改；  
先启动shadowsocks-netty  
然后运行ClientExecuteSOCKS

1.结果报如下错误：

```
I/O exception (org.apache.http.NoHttpResponseException) caught when processing request to {}->http://www.google.com:80: 
The target server failed to respond
```

可以观察shadowsocks-netty的服务器端[shadowsocks-netty-server](https://gitee.com/OutOfMemory/shadowsocks-netty-server)，有如下日志：

```
org.netty.proxy.ClientProxyHandler$2 - connect fail host = 67.15.129.210,port = 80,inetAddress = /67.15.129.210
```

域名解析后的ip地址连接失败，多次试验ip地址是会变动的，导致有时候能成功，有时候失败；

针对此问题可以直接使用域名访问，代码做如下修改：

```
sock.connect(remoteAddress, connectTimeout);
```

将如上代码改成：

```
sock.connect(InetSocketAddress.createUnresolved(remoteAddress.getHostName(), remoteAddress.getPort()),connectTimeout);
```

2.重新运行，报如下错误：

```
Caused by: org.apache.http.ProtocolException: The server failed to respond with a valid HTTP response
    at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:151)
    at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:57)
    at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:260)
    at org.apache.http.impl.DefaultBHttpClientConnection.receiveResponseHeader(DefaultBHttpClientConnection.java:161)
    at org.apache.http.impl.conn.CPoolProxy.receiveResponseHeader(CPoolProxy.java:153)
    at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:271)
    at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:123)
    at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:254)
    at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:195)
    at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:86)
    at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:108)
    at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:184)
    ... 2 more
```

通过debug进入DefaultHttpResponseParser的parseHead方法中发现，每次读取http协议的状态行是” HTTP/1.1 200 OK”，有两个空格cunz，  
导致比对失败，经分析发现是在Socks四次握手的时候没有将握手数据读取干净，导致后面的真实数据出现脏数据；  
分别看netty提供的SocksCmdResponse和jdk中的SocksSocketImpl类：

```
public void encodeAsByteBuf(ByteBuf byteBuf) {
        byteBuf.writeByte(protocolVersion().byteValue());
        byteBuf.writeByte(cmdStatus.byteValue());
        byteBuf.writeByte(0x00);
        byteBuf.writeByte(addressType.byteValue());
        switch (addressType) {
            case IPv4: {
                byte[] hostContent = host == null ?
                        IPv4_HOSTNAME_ZEROED : NetUtil.createByteArrayFromIpAddressString(host);
                byteBuf.writeBytes(hostContent);
                byteBuf.writeShort(port);
                break;
            }
            case DOMAIN: {
                byte[] hostContent = host == null ?
                        DOMAIN_ZEROED : host.getBytes(CharsetUtil.US_ASCII);
                byteBuf.writeByte(hostContent.length);   // domain length
                byteBuf.writeBytes(hostContent);   // domain value
                byteBuf.writeShort(port);  // port value
                break;
            }
            case IPv6: {
                byte[] hostContent = host == null
                        ? IPv6_HOSTNAME_ZEROED : NetUtil.createByteArrayFromIpAddressString(host);
                byteBuf.writeBytes(hostContent);
                byteBuf.writeShort(port);
                break;
            }
        }
    }
```

SocksSocketImpl类connect方法部分代码如下：

```
switch (data[1]) {
        case REQUEST_OK:
            // success!
            switch(data[3]) {
            case IPV4:
                addr = new byte[4];
                i = readSocksReply(in, addr, deadlineMillis);
                if (i != 4)
                    throw new SocketException("Reply from SOCKS server badly formatted");
                data = new byte[2];
                i = readSocksReply(in, data, deadlineMillis);
                if (i != 2)
                    throw new SocketException("Reply from SOCKS server badly formatted");
                break;
            case DOMAIN_NAME:
                len = data[1];
                byte[] host = new byte[len];
                i = readSocksReply(in, host, deadlineMillis);
                if (i != len)
                    throw new SocketException("Reply from SOCKS server badly formatted");
                data = new byte[2];
                i = readSocksReply(in, data, deadlineMillis);
                if (i != 2)
                    throw new SocketException("Reply from SOCKS server badly formatted");
                break;
        ......
}
```

shadowsocks-netty返回的addressType为DOMAIN类型，会发现写入的数据格式和读取的格式不一致，导致产生脏数据；

此问题可以修改shadowsocks-netty返回的addressType为IPV4类型，具体代码在SocksServerConnectHandler中：

```
private SocksCmdResponse getSuccessResponse(SocksCmdRequest request) {
    return new SocksCmdResponse(SocksCmdStatus.SUCCESS, SocksAddressType.IPv4);
}
```

修改之后运行正确结果如下：

```
HTTP/1.1 200 OK
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en"><head><meta content="Search the world's ...具体网页内容省略...</body></html>

```

**下载视频**  
下载视频的部分代码如下：

```
public class ClientExecuteSOCKS2 {
 
    /** 代理参数 IP+PORT **/
    private static String PROXY_IP = "localhost";
    private static int PROXY_PORT = 1080;
 
    public static void main(String[] args) throws Exception {
        Registry<ConnectionSocketFactory> reg = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("http", new MyConnectionSocketFactory()).build();
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(reg);
        CloseableHttpClient httpclient = HttpClients.custom().setConnectionManager(cm).build();
        try {
            InetSocketAddress socksaddr = new InetSocketAddress(PROXY_IP, PROXY_PORT);
            HttpClientContext context = HttpClientContext.create();
            context.setAttribute("socks.address", socksaddr);
            HttpGet request = new HttpGet("http://xxxxxxx.mp4");
            CloseableHttpResponse response = httpclient.execute(request, context);
 
            InputStream is = null;
            OutputStream os = null;
            try {
                System.out.println(response.getStatusLine());
                is = response.getEntity().getContent();
                System.out.println(response.getEntity().getContentLength());
                os = new FileOutputStream(new File("D:\\tmp.mp4"));
                byte tmp[] = new byte[1024];
                int l;
                while ((l = is.read(tmp)) != -1) {
                    os.write(tmp, 0, l);
                }
                os.flush();
            } finally {
                if (response != null) {
                    response.close();
                }
                if (is != null) {
                    is.close();
                }
                if (os != null) {
                    os.close();
                }
            }
        } finally {
            httpclient.close();
        }
    }
}
```

**总结**  
下载国外视频有很多种方式，比如浏览器插件，本文依赖客户端Socks5代理程序，使用Httpclient4进行资源下载，更容易自动化和可控性；本文主要用于学习使用。

**个人博客：[codingo.xyz](http://codingo.xyz/)**