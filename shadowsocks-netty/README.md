##《Netty实现shadowsocks客户端》

## Shadowsocks是什么

shadowsocks是基于socks5协议实现的代理方式，分为服务器和客户端，双端之间通过使用指定的加密方式(AES,BlowFish,Table等)进行数据传输，有效的突破了GFW。

整个流程可以看来自网上的一张流程图：

![](http://static.oschina.net/uploads/space/2016/0908/141353_x7b2_159239.png)

总体流程大致如上图所示，具体实现细节后面会介绍，今天要实现的是SS Local这个模块，数据传输都是基于socks5协议，所以有必要先了解一下socks5协议。

## socks5协议

SOCKS5 是一个代理协议，它在使用TCP/IP协议通讯的前端机器和服务器机器之间扮演一个中介角色，使得内部网中的前端机器变得能够访问Internet网中的服务器，或者使通讯更加安全。SOCKS5 服务器通过将前端发来的请求转发给真正的目标服务器， 模拟了一个前端的行为。

**1.**建立与SOCKS5服务器的TCP连接后客户端需要先发送请求来协商版本及认证方式，格式为（以字节为单位）：

```java
+-----+----------+-----------+
| VER | NMETHODS |  METHODS  |
+-----+----------+-----------+
|  1  |    1     |  1 to 255 |
+-----+----------+-----------+
```

VER是SOCKS版本，这里应该是0x05；

NMETHODS是METHODS部分的长度；

METHODS是客户端支持的认证方式列表，每个方法占1字节。当前的定义是：

```java
0x00  不需要认证
0x01  GSSAPI
0x02  用户名、密码认证
0x03  - 0x7F由IANA分配（保留）
0x80  - 0xFE为私人方法保留
0xFF  无可接受的方法
```

**2.**服务器从客户端提供的方法中选择一个并通过以下消息通知客户端（以字节为单位）：

```java
+-----+---------+
| VER | NMETHOD |
+-----+---------+
|  1  |    1    |
+-----+---------+
```

VER是SOCKS版本，这里应该是0x05；

METHOD是服务端选中的方法。如果返回0xFF表示没有一个认证方法被选中，客户端需要关闭连接。

之后客户端和服务端根据选定的认证方式执行对应的认证。

**3.**认证结束后客户端就可以发送请求信息。如果认证方法有特殊封装要求，请求必须按照方法所定义的方式进行封装。

SOCKS5请求格式（以字节为单位）：

```
+-----+-----+-------+------+----------+----------+
| VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
+-----+-----+-------+------+----------+----------+
|  1  |  1  | X'00' |  1   | Variable |    2     |
+-----+-----+-------+------+----------+----------+
```

VER是SOCKS版本，这里应该是0x05；

CMD是SOCK的命令码

```
0x01表示CONNECT请求
0x02表示BIND请求
0x03表示UDP转发
```

RSV 0x00，保留

ATYP 后面的地址类型

```
0x01 IPv4地址，DST ADDR部分4字节长度
0x03 域名，DST ADDR部分第一个字节为域名长度，DST ADDR剩余的内容为域名，没有\0结尾。
0x04 IPv6地址，16个字节长度。
```

DST.ADDR 目的地址

DST.PROT 目的端口

**4.**服务器按以下格式回应客户端的请求（以字节为单位）：

```java
+-----+-----+-------+------+----------+----------+
| VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
+-----+-----+-------+------+----------+----------+
|  1  |  1  | X'00' |  1   | Variable |    2     |
+-----+-----+-------+------+----------+----------+
```

REP 应答字段

```
0x00 表示成功
0x01 普通SOCKS服务器连接失败
0x02 现有规则不允许连接
0x03 网络不可达
0x04 主机不可达
0x05 连接被拒
0x06 TTL超时
0x07 不支持的命令
0x08 不支持的地址类型
0x09 0xFF未定义
```

BND.ADDR 服务器绑定的地址  
BND.PORT 以网络字节顺序表示的服务器绑定的端口

更多的socks5介绍[rfc1928](https://www.ietf.org/rfc/rfc1928.txt)

四次握手结束之后，client 会把代理服务器当作是自己真正想通讯的服务器一样对待，这时代理服务器才会转发真正的数据。

## 具体实现

根据上面整体的流程图以及对socks5的了解，具体实现整理了如下几步：

```
1.pc向ss local发送请求来协商版本和认证方法，ss local回复pc
2.pc发送详细的请求，并且指定的cmdType为connect
ss local处理cmdType为connect的请求；
此时ss local和ss server建立连接，如果连接成功ss local回复pc为成功否则为失败；
如果成功ss local发送请求给ss server，发送的内容是pc发送给ss local的内容跳过前面的3个字节
3.握手成功后需要为pc和ss local建立的channel添加消息处理器，用来接受pc的请求，并进行加密转发给ss server；
同时为ss local和ss server建立的channle添加消息处理器，用来接受ss server的请求，并进行解密转发给pc
4.配置浏览器，进行测试
```

因为netty的高性能以及4.0以后对socks5协议的支持，这里选择了：netty4.0.41

```
<dependency>
	<groupId>io.netty</groupId>
	<artifactId>netty-all</artifactId>
	<version>4.0.41.Final</version>
</dependency>
```

以下代码参考了netty4.0.41自带的example中的socksproxy

### 第一步：处理协商版本和认证方法

因为netty4.0.x对socks5协议的支持，直接使用SocksInitRequestDecoder和SocksMessageEncoder

```
public final class SocksServerInitializer extends
		ChannelInitializer<SocketChannel> {

	private SocksMessageEncoder socksMessageEncoder;
	private SocksServerHandler socksServerHandler;

	public SocksServerInitializer(Config config) {
		socksMessageEncoder = new SocksMessageEncoder();
		socksServerHandler = new SocksServerHandler(config);
	}

	@Override
	public void initChannel(SocketChannel socketChannel) throws Exception {
		ChannelPipeline p = socketChannel.pipeline();
		p.addLast(new SocksInitRequestDecoder());
		p.addLast(socksMessageEncoder);
		p.addLast(socksServerHandler);
	}
}
```

经过SocksInitRequestDecoder的解码，数据流转到自定义的SocksServerHandler中

```
@ChannelHandler.Sharable
public final class SocksServerHandler extends
		SimpleChannelInboundHandler<SocksRequest> {

	private static Log logger = LogFactory.getLog(SocksServerHandler.class);

	private Config config;

	public SocksServerHandler(Config config) {
		this.config = config;
	}

	@Override
	public void channelRead0(ChannelHandlerContext ctx,
			SocksRequest socksRequest) throws Exception {
		switch (socksRequest.requestType()) {
		case INIT: {
			logger.info("localserver init");
			ctx.pipeline().addFirst(new SocksCmdRequestDecoder());
			ctx.write(new SocksInitResponse(SocksAuthScheme.NO_AUTH));
			break;
		}
		case AUTH:
			ctx.pipeline().addFirst(new SocksCmdRequestDecoder());
			ctx.write(new SocksAuthResponse(SocksAuthStatus.SUCCESS));
			break;
		case CMD:
			SocksCmdRequest req = (SocksCmdRequest) socksRequest;
			if (req.cmdType() == SocksCmdType.CONNECT) {
				logger.info("localserver connect");
				ctx.pipeline().addLast(new SocksServerConnectHandler(config));
				ctx.pipeline().remove(this);
				ctx.fireChannelRead(socksRequest);
			} else {
				ctx.close();
			}
			break;
		case UNKNOWN:
			ctx.close();
			break;
		}
	}

	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) {
		ctx.flush();
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable throwable) {
		throwable.printStackTrace();
		SocksServerUtils.closeOnFlush(ctx.channel());
	}
}

```

对pc发送的协议版本和认证方法，SocksInitRequestDecoder将它解码为请求类型为INIT，pc和ss local的连接我们是不需要验证的，所有这里返回的是SocksAuthScheme.NO_AUTH，十六进制是：0x00，可以对照前面介绍的socks5协议。同时也添加了SocksCmdRequestDecoder，用来解码接下来的请求协议。

### 第二步：处理详细的请求，完成握手

经过SocksCmdRequestDecoder的解码，请求类型变成了CMD，同时处理cmdType为connect；此时添加了SocksServerConnectHandler，并将SocksServerHandler移除，这样握手成功后发送的数据都不经过SocksServerHandler了，交给SocksServerConnectHandler处理。

```
@ChannelHandler.Sharable
public final class SocksServerConnectHandler extends
		SimpleChannelInboundHandler<SocksCmdRequest> {

	private static Log logger = LogFactory
			.getLog(SocksServerConnectHandler.class);
	
	public static final int BUFFER_SIZE = 16384;

	private final Bootstrap b = new Bootstrap();
	private ICrypt _crypt;
	private ByteArrayOutputStream _remoteOutStream;
	private ByteArrayOutputStream _localOutStream;
	private Config config;

	public SocksServerConnectHandler(Config config) {
		this.config = config;
		this._crypt = CryptFactory.get(config.get_method(), config.get_password());
		this._remoteOutStream = new ByteArrayOutputStream(BUFFER_SIZE);
		this._localOutStream = new ByteArrayOutputStream(BUFFER_SIZE);
	}

	@Override
	public void channelRead0(final ChannelHandlerContext ctx,
			final SocksCmdRequest request) throws Exception {
		Promise<Channel> promise = ctx.executor().newPromise();
		promise.addListener(new GenericFutureListener<Future<Channel>>() {
			@Override
			public void operationComplete(final Future<Channel> future)
					throws Exception {
				final Channel outboundChannel = future.getNow();
				if (future.isSuccess()) {
					final InRelayHandler inRelay = new InRelayHandler(ctx
							.channel(), SocksServerConnectHandler.this);
					final OutRelayHandler outRelay = new OutRelayHandler(
							outboundChannel, SocksServerConnectHandler.this);

					ctx.channel().writeAndFlush(getSuccessResponse(request))
							.addListener(new ChannelFutureListener() {
								@Override
								public void operationComplete(
										ChannelFuture channelFuture) {
									sendConnectRemoteMessage(request, outboundChannel);
									
									ctx.pipeline().remove(SocksServerConnectHandler.this);
									outboundChannel.pipeline().addLast(inRelay);
									ctx.pipeline().addLast(outRelay);
								}
							});
				} else {
					ctx.channel().writeAndFlush(
							new SocksCmdResponse(SocksCmdStatus.FAILURE,
									request.addressType()));
					SocksServerUtils.closeOnFlush(ctx.channel());
				}
			}
		});
		
		final Channel inboundChannel = ctx.channel();
		b.group(inboundChannel.eventLoop()).channel(NioSocketChannel.class)
				.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
				.option(ChannelOption.SO_KEEPALIVE, true)
				.handler(new DirectClientHandler(promise));

		b.connect(config.get_ipAddr(), config.get_port()).addListener(
				new ChannelFutureListener() {
					@Override
					public void operationComplete(ChannelFuture future)
							throws Exception {
						if (!future.isSuccess()) {
							ctx.channel().writeAndFlush(getFailureResponse(request));
							SocksServerUtils.closeOnFlush(ctx.channel());
						}
					}
				});
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
			throws Exception {
		SocksServerUtils.closeOnFlush(ctx.channel());
	}
}
```

首先ss local和ss server建立了连接，根据连接成功还是失败ss local回复pc，分别是SocksCmdStatus.SUCCESS（0x00）和SocksCmdStatus.FAILURE（0x01），并且在成功之后向ss server发送请求

```
	private void sendConnectRemoteMessage(SocksCmdRequest request,
			Channel outboundChannel) {
		ByteBuf buff = Unpooled.directBuffer();
		request.encodeAsByteBuf(buff);
		if (!buff.hasArray()) {
			int len = buff.readableBytes();
			byte[] arr = new byte[len];
			buff.getBytes(0, arr);
			byte[] data = remoteByte(arr);
			sendRemote(data, data.length, outboundChannel);
		}
	}
```

请求其实就是pc发送给ss local的消息，只不过需要跳过前面的3个字节

```
	public void sendRemote(byte[] data, int length, Channel channel) {
		_crypt.encrypt(data, length, _remoteOutStream);
		byte[] sendData = _remoteOutStream.toByteArray();
		channel.writeAndFlush(Unpooled.wrappedBuffer(sendData));
		logger.info("sendRemote message:length = " + length+",channel = " + channel);
	}
```

发送给ss server需要对消息进行加密_crypt.encrypt()

### 第三步：为2个channel添加消息处理器

2个channel分别是pc->ss local 和 ss local-> ss server

SocksServerConnectHandler中在和ss server连接成功之后移除了SocksServerConnectHandler，以后pc发送给ss local的消息都不在经过它，交给OutRelayHandler，同时为ss local-> ss server的channel添加了InRelayHandler处理器

```
public final class OutRelayHandler extends ChannelInboundHandlerAdapter {

	private final Channel relayChannel;
	private SocksServerConnectHandler connectHandler;

	public OutRelayHandler(Channel relayChannel,
			SocksServerConnectHandler connectHandler) {
		this.relayChannel = relayChannel;
		this.connectHandler = connectHandler;
	}

	@Override
	public void channelActive(ChannelHandlerContext ctx) {
		ctx.writeAndFlush(Unpooled.EMPTY_BUFFER);
	}

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
		if (relayChannel.isActive()) {
			ByteBuf bytebuff = (ByteBuf) msg;
			if (!bytebuff.hasArray()) {
				int len = bytebuff.readableBytes();
				byte[] arr = new byte[len];
				bytebuff.getBytes(0, arr);
				connectHandler.sendRemote(arr, arr.length, relayChannel);
			}
		} else {
			ReferenceCountUtil.release(msg);
		}
	}

	@Override
	public void channelInactive(ChannelHandlerContext ctx) {
		if (relayChannel.isActive()) {
			SocksServerUtils.closeOnFlush(relayChannel);
		}
	}

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
		cause.printStackTrace();
		ctx.close();
	}

}
```

OutRelayHandler接受到pc的消息之后，对消息加密，同时转发给ss server

InRelayHandler接受到pc的消息之后，对消息解密，同时转发给ss local

整个流程大概就是这样，以上代码参考了一些开源项目，比如加密解密那块

### 第四步：配置浏览器进行测试

chrome浏览器本身是不支持socks5的，需要安装Proxy SwitchySharp插件

Firefox浏览器本身是支持socks5的

配置：

指定socks5协议，指定ip为：127.0.0.1，端口为：配置文件中配置的localport，默认是1081

启动local server：

![](http://static.oschina.net/uploads/space/2016/0908/182625_o6T4_159239.jpg)

指定浏览器的代理配置：

![](http://static.oschina.net/uploads/space/2016/0908/182802_X8h5_159239.jpg)

正常打开Google，Facebook，并且流畅的看完了一集youtube上的视频

## 源码

**github：[https://github.com/ksfzhaohui/shadowsocks-netty](https://github.com/ksfzhaohui/shadowsocks-netty)**

**oscgit：[http://git.oschina.net/OutOfMemory/shadowsocks-netty](http://git.oschina.net/OutOfMemory/shadowsocks-netty)**