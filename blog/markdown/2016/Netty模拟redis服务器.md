Redis的客户端与服务端采用叫做  **RESP(Redis Serialization Protocol)**的网络通信协议交换数据，客户端和服务器通过 TCP 连接来进行数据交互， 服务器默认的端口号为 6379 。客户端和服务器发送的命令或数据一律以 **\\r\\n （CRLF）**结尾。

RESP支持五种数据类型：

状态回复（status reply）：以“+”开头，表示正确的状态信息，”+”后就是具体信息​，比如：

```
redis 127.0.0.1:6379> set ss sdf
OK
```

其实它真正回复的数据是：+OK\\r\\n  
错误回复（error reply）：以”-“开头，表示错误的状态信息，”-“后就是具体信息，比如：

```
redis 127.0.0.1:6379> incr ss
(error) ERR value is not an integer or out of range
```

整数回复（integer reply）：以”:”开头，表示对某些操作的回复比如DEL, EXISTS, INCR等等

```
redis 127.0.0.1:6379> incr aa
(integer) 1
```

批量回复（bulk reply）：以”$”开头，表示下一行的字符串长度，具体字符串在下一行中

多条批量回复（multi bulk reply）：以”*”开头，表示消息体总共有多少行（不包括当前行）”*”是具体行数

```
redis 127.0.0.1:6379> get ss
"sdf"

客户端->服务器
*2\r\n
$3\r\n
get\r\n
$2\r\n
ss\r\n
服务器->客户端
$3\r\n
sdf\r\n
```

注：以上写的都是XX回复，并不是说协议格式只是适用于服务器->客户端，客户端->服务器端也同样使用以上协议格式，其实双端协议格式的统一更加方便扩展

回到正题，我们这里是通过netty来模拟redis服务器，可以整理一下思路大概分为这么几步：

```
1.需要一个底层的通信框架，这里选择的是netty4.0.25
2.需要对客户端穿过来的数据进行解码(Decoder)，其实就是分别处理以上5种数据类型
3.解码以后我们封装成更加利于理解的命令(Command)，比如：set<name> foo hello<params>
4.有了命令以后就是处理命令(execute)，其实我们可以去连接正在的redis服务器，不过这里只是简单的模拟
5.处理完之后就是封装回复(Reply)，然后编码(Encoder),需要根据不同的命令分别返回以后5种数据类型
6.测试验证，通过redis-cli去连接netty模拟的redis服务器，看能否返回正确的结果
```

以上思路参考github上的一个项目：[https://github.com/spullara/redis-protocol](https://github.com/spullara/redis-protocol)，测试代码也是在此基础上做了一个简化

**第一步：通信框架netty**

```
<dependency>
	<groupId>io.netty</groupId>
	<artifactId>netty-all</artifactId>
	<version>4.0.25.Final</version>
</dependency>
```

**第二步：数据类型解码**

```
public class RedisCommandDecoder extends ReplayingDecoder<Void> {

	public static final char CR = '\r';
	public static final char LF = '\n';

	public static final byte DOLLAR_BYTE = '$';
	public static final byte ASTERISK_BYTE = '*';

	private byte[][] bytes;
	private int arguments = 0;

	@Override
	protected void decode(ChannelHandlerContext ctx, ByteBuf in,
			List<Object> out) throws Exception {
		if (bytes != null) {
			int numArgs = bytes.length;
			for (int i = arguments; i < numArgs; i++) {
				if (in.readByte() == DOLLAR_BYTE) {
					int l = RedisReplyDecoder.readInt(in);
					if (l > Integer.MAX_VALUE) {
						throw new IllegalArgumentException(
								"Java only supports arrays up to "
										+ Integer.MAX_VALUE + " in size");
					}
					int size = (int) l;
					bytes[i] = new byte[size];
					in.readBytes(bytes[i]);
					if (in.bytesBefore((byte) CR) != 0) {
						throw new RedisException("Argument doesn't end in CRLF");
					}
					// Skip CRLF(\r\n)
					in.skipBytes(2);
					arguments++;
					checkpoint();
				} else {
					throw new IOException("Unexpected character");
				}
			}
			try {
				out.add(new Command(bytes));
			} finally {
				bytes = null;
				arguments = 0;
			}
		} else if (in.readByte() == ASTERISK_BYTE) {
			int l = RedisReplyDecoder.readInt(in);
			if (l > Integer.MAX_VALUE) {
				throw new IllegalArgumentException(
						"Java only supports arrays up to " + Integer.MAX_VALUE
								+ " in size");
			}
			int numArgs = (int) l;
			if (numArgs < 0) {
				throw new RedisException("Invalid size: " + numArgs);
			}
			bytes = new byte[numArgs][];
			checkpoint();
			decode(ctx, in, out);
		} else {
			in.readerIndex(in.readerIndex() - 1);
			byte[][] b = new byte[1][];
			b[0] = in.readBytes(in.bytesBefore((byte) CR)).array();
			in.skipBytes(2);
			out.add(new Command(b, true));
		}
	}

}
```

首先通过接受到以“*”开头的多条批量类型初始化二维数组byte\[\]\[\] bytes，以读取到第一个以\\r\\n结尾的数据作为数组的长度，然后再处理以“$”开头的批量类型。  
以上除了处理我们熟悉的批量和多条批量类型外，还处理了没有任何标识的数据，其实有一个专门的名字叫**Inline命令：**  
有些时候仅仅是telnet连接Redis服务，或者是仅仅向Redis服务发送一个命令进行检测。虽然Redis协议可以很容易的实现，但是使用Interactive sessions 并不理想，而且redis-cli也不总是可以使用。基于这些原因，Redis支持特殊的命令来实现上面描述的情况。这些命令的设计是很人性化的，被称作Inline 命令。

**第三步：封装command对象**

由第二步中可以看到不管是commandName还是params都统一放在了字节二维数组里面，最后封装在command对象里面

```
public class Command {
	public static final byte[] EMPTY_BYTES = new byte[0];

	private final Object name;
	private final Object[] objects;
	private final boolean inline;

	public Command(Object[] objects) {
		this(null, objects, false);
	}

	public Command(Object[] objects, boolean inline) {
		this(null, objects, inline);
	}

	private Command(Object name, Object[] objects, boolean inline) {
		this.name = name;
		this.objects = objects;
		this.inline = inline;
	}

	public byte[] getName() {
		if (name != null)
			return getBytes(name);
		return getBytes(objects[0]);
	}

	public boolean isInline() {
		return inline;
	}

	private byte[] getBytes(Object object) {
		byte[] argument;
		if (object == null) {
			argument = EMPTY_BYTES;
		} else if (object instanceof byte[]) {
			argument = (byte[]) object;
		} else if (object instanceof ByteBuf) {
			argument = ((ByteBuf) object).array();
		} else if (object instanceof String) {
			argument = ((String) object).getBytes(Charsets.UTF_8);
		} else {
			argument = object.toString().getBytes(Charsets.UTF_8);
		}
		return argument;
	}

	public void toArguments(Object[] arguments, Class<?>[] types) {
		for (int position = 0; position < types.length; position++) {
			if (position >= arguments.length) {
				throw new IllegalArgumentException(
						"wrong number of arguments for '"
								+ new String(getName()) + "' command");
			}
			if (objects.length - 1 > position) {
				arguments[position] = objects[1 + position];
			}
		}
	}

}
```

所有的数据都放在了Object数组里面，而且可以通过getName方法知道Object\[0\]就是commandName

**第四步：执行命令**

在经历了解码和封装之后，下面需要实现handler类，用来处理消息

```
public class RedisCommandHandler extends SimpleChannelInboundHandler<Command> {

	private Map<String, Wrapper> methods = new HashMap<String, Wrapper>();

	interface Wrapper {
		Reply<?> execute(Command command) throws RedisException;
	}

	public RedisCommandHandler(final RedisServer rs) {
		Class<? extends RedisServer> aClass = rs.getClass();
		for (final Method method : aClass.getMethods()) {
			final Class<?>[] types = method.getParameterTypes();
			methods.put(method.getName(), new Wrapper() {
				@Override
				public Reply<?> execute(Command command) throws RedisException {
					Object[] objects = new Object[types.length];
					try {
						command.toArguments(objects, types);
						return (Reply<?>) method.invoke(rs, objects);
					} catch (Exception e) {
						return new ErrorReply("ERR " + e.getMessage());
					}
				}
			});
		}
	}

	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		ctx.flush();
	}

	@Override
	protected void channelRead0(ChannelHandlerContext ctx, Command msg)
			throws Exception {
		String name = new String(msg.getName());
		Wrapper wrapper = methods.get(name);
		Reply<?> reply;
		if (wrapper == null) {
			reply = new ErrorReply("unknown command '" + name + "'");
		} else {
			reply = wrapper.execute(msg);
		}
		if (reply == StatusReply.QUIT) {
			ctx.close();
		} else {
			if (msg.isInline()) {
				if (reply == null) {
					reply = new InlineReply(null);
				} else {
					reply = new InlineReply(reply.data());
				}
			}
			if (reply == null) {
				reply = ErrorReply.NYI_REPLY;
			}
			ctx.write(reply);
		}
	}
}
```

在实例化handler的时候传入了一个RedisServer对象，这个方法是真正用来处理redis命令的，理论上这个对象应该支持redis的所有命令，不过这里只是测试所有只提供了2个方法：

```
public interface RedisServer {

	public BulkReply get(byte[] key0) throws RedisException;

	public StatusReply set(byte[] key0, byte[] value1) throws RedisException;

}
```

在channelRead0方法中我们可以拿到之前封装好的command方法，然后通过命令名称执行操作，这里的RedisServer也很简单，只是用简单的hashmap进行临时的保存数据。

**第五步：封装回复**

第四步种我们可以看到处理完命令之后，返回了一个Reply对象

```
public interface Reply<T> {

	byte[] CRLF = new byte[] { RedisReplyDecoder.CR, RedisReplyDecoder.LF };

	T data();

	void write(ByteBuf os) throws IOException;
}
```

根据上面提到的5种类型再加上一个inline命令，根据不同的数据格式进行拼接，比如StatusReply：

```
public void write(ByteBuf os) throws IOException {
	os.writeByte('+');
	os.writeBytes(statusBytes);
	os.writeBytes(CRLF);
}
```

所以对应Decoder的Encoder就很简单了：

```
public class RedisReplyEncoder extends MessageToByteEncoder<Reply<?>> {
	@Override
	public void encode(ChannelHandlerContext ctx, Reply<?> msg, ByteBuf out)
			throws Exception {
		msg.write(out);
	}
}

```

只需要将封装好的Reply返回给客户端就行了

**最后一步：测试**

启动类：

```
public class Main {
	private static Integer port = 6379;

	public static void main(String[] args) throws InterruptedException {
		final RedisCommandHandler commandHandler = new RedisCommandHandler(
				new SimpleRedisServer());

		ServerBootstrap b = new ServerBootstrap();
		final DefaultEventExecutorGroup group = new DefaultEventExecutorGroup(1);
		try {
			b.group(new NioEventLoopGroup(), new NioEventLoopGroup())
					.channel(NioServerSocketChannel.class)
					.option(ChannelOption.SO_BACKLOG, 100).localAddress(port)
					.childOption(ChannelOption.TCP_NODELAY, true)
					.childHandler(new ChannelInitializer<SocketChannel>() {
						@Override
						public void initChannel(SocketChannel ch)
								throws Exception {
							ChannelPipeline p = ch.pipeline();
							p.addLast(new RedisCommandDecoder());
							p.addLast(new RedisReplyEncoder());
							p.addLast(group, commandHandler);
						}
					});

			ChannelFuture f = b.bind().sync();
			f.channel().closeFuture().sync();
		} finally {
			group.shutdownGracefully();
		}
	}
}
```

ChannelPipeline分别添加了RedisCommandDecoder、RedisReplyEncoder和RedisCommandHandler，同时我们启动的端口和Redis服务器端口是一样的也是**6379**

打开redis-cli程序：

```
redis 127.0.0.1:6379> get dsf
(nil)
redis 127.0.0.1:6379> set dsf dsfds
OK
redis 127.0.0.1:6379> get dsf
"dsfds"
redis 127.0.0.1:6379>
```

从结果可以看出和正常使用redis服务器没有差别

**总结**

这样做的意义其实就是可以把它当做一个**redis代理**，由这个代理服务器去进行sharding处理，客户端不直接访问redis服务器，对客户端来说，后台redis集群是完全透明的。

**个人博客：[codingo.xyz](http://codingo.xyz/)**