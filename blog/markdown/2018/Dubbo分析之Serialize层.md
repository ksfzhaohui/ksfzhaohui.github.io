##Dubbo�������
����Dubbo��������ƿ��Բ鿴�ٷ��ĵ�����ͼ���������ı��Dubbo��������ƣ�
![ͼƬ����][1]

###1.ͼ��˵��
ͼ����ߵ���������Ϊ�������ѷ�ʹ�õĽӿڣ��ұߵ���ɫ������Ϊ�����ṩ��ʹ�õĽӿڣ�λ���������ϵ�Ϊ˫�����õ��Ľӿڣ�
ͼ�д������Ϸ�Ϊʮ�㣬�����Ϊ�����������ұߵĺ�ɫ��ͷ�����֮���������ϵ��
ͼ����ɫС���Ϊ��չ�ӿڣ���ɫС��Ϊʵ���࣬ͼ��ֻ��ʾ���ڹ��������ʵ���ࣻ
ͼ����ɫ����Ϊ��ʼ�����̣�������ʱ��װ������ɫʵ��Ϊ�������ù��̣�������ʱ��ʱ������ɫ���Ǽ�ͷΪ�̳У����԰����࿴�������ͬһ���ڵ㣬���ϵ�����Ϊ���õķ�����

###2.����˵��
config ���ò㣺�������ýӿڣ��� ServiceConfig, ReferenceConfig Ϊ���ģ�����ֱ�ӳ�ʼ�������࣬Ҳ����ͨ�� spring �����������������ࣻ
proxy �������㣺����ӿ�͸���������ɷ���Ŀͻ��� Stub �ͷ������� Skeleton, �� ServiceProxy Ϊ���ģ���չ�ӿ�Ϊ ProxyFactory��
registry ע�����Ĳ㣺��װ�����ַ��ע���뷢�֣��Է��� URL Ϊ���ģ���չ�ӿ�Ϊ RegistryFactory, Registry, RegistryService��
cluster ·�ɲ㣺��װ����ṩ�ߵ�·�ɼ����ؾ��⣬���Ž�ע�����ģ��� Invoker Ϊ���ģ���չ�ӿ�Ϊ Cluster, Directory, Router, LoadBalance��
monitor ��ز㣺RPC ���ô����͵���ʱ���أ��� Statistics Ϊ���ģ���չ�ӿ�Ϊ MonitorFactory, Monitor, MonitorService��
protocol Զ�̵��ò㣺��װ RPC ���ã��� Invocation, Result Ϊ���ģ���չ�ӿ�Ϊ Protocol, Invoker, Exporter��
exchange ��Ϣ�����㣺��װ������Ӧģʽ��ͬ��ת�첽���� Request, Response Ϊ���ģ���չ�ӿ�Ϊ Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer��
transport ���紫��㣺���� mina �� netty Ϊͳһ�ӿڣ��� Message Ϊ���ģ���չ�ӿ�Ϊ Channel, Transporter, Client, Server, Codec��
serialize �������л��㣺�ɸ��õ�һЩ���ߣ���չ�ӿ�Ϊ Serialization, ObjectInput, ObjectOutput, ThreadPool��

���Ľ�����ײ��serialize�㿪ʼ����dubbo����Դ�������

##ͨѶ���
dubbo�ĵײ�ͨѶʹ�õ��ǵ�������ܣ�������netty��netty4��mina��grizzly��Ĭ��ʹ�õ���netty���ֱ��ṩ��server��(�����ṩ��)��client��(�������ѷ�)��������ʹ�õ�nettyΪ��������һ��NettyServer�Ĳ��ִ��룺

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
�����������ṩ��ʱ�ͻ���ô�doOpen������������������˿ڣ������ѷ����ӣ����ϴ�����ǳ��������nettyServer�˴��룬��Ϊ�����ص����dubbo�����л�������������Ҫ��decoder��encoder����������ֱ�����NettyCodecAdapter�У�

```
private final ChannelHandler encoder = new InternalEncoder();
private final ChannelHandler decoder = new InternalDecoder();
```
###1.������
��NettyCodecAdapter�������ڲ���InternalEncoder��

```
private class InternalEncoder extends OneToOneEncoder {
 
        @Override
        protected Object encode(ChannelHandlerContext ctx, Channel ch, Object msg) throws Exception {
            com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer =
                    com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(1024);
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            try {
                codec.encode(channel, buffer, msg);
            } finally {
                NettyChannel.removeChannelIfDisconnected(ch);
            }
            return ChannelBuffers.wrappedBuffer(buffer.toByteBuffer());
        }
    }
```
������ʵ�Ƕ�codec�İ�װ������û�������봦�������ص㿴һ��codec�࣬������һ���ӿ��࣬�ж���ʵ���࣬Codec2Դ�����£�

```
@SPI
public interface Codec2 {
 
    @Adaptive({Constants.CODEC_KEY})
    void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException;
 
    @Adaptive({Constants.CODEC_KEY})
    Object decode(Channel channel, ChannelBuffer buffer) throws IOException;
 
 
    enum DecodeResult {
        NEED_MORE_INPUT, SKIP_SOME_INPUT
    }
 
}
```
ʵ�ְ�����TransportCodec��TelnetCodec��ExchangeCodec��DubboCountCodec�Լ�ThriftCodec����ȻҲ����������չ������������ʱ��ÿ�����Ͷ����أ�dubbo��ͨ���������ļ������ú����е����ͣ�Ȼ������������Ҫʲô�����ʲô�࣬
�����ļ��ľ���·����META-INF/dubbo/internal/com.alibaba.dubbo.remoting.Codec2���������£�

```
transport=com.alibaba.dubbo.remoting.transport.codec.TransportCodec
telnet=com.alibaba.dubbo.remoting.telnet.codec.TelnetCodec
exchange=com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec
thrift=com.alibaba.dubbo.rpc.protocol.thrift.ThriftCodec
```
��ȡ����Codec2�Ĵ������£�

```
protected static Codec2 getChannelCodec(URL url) {
    String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");
    if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) {
        return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
    } else {
        return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class)
                .getExtension(codecName));
    }
}
```
ͨ����url�л�ȡ�Ƿ��йؼ���codec������еĻ��ͻ�ȡ��ǰ��ֵ��dubboĬ�ϵ�codecΪdubbo�����û��ֵĬ��Ϊtelnet��������Ĭ��ֵΪdubbo������ʵ����DubboCountCodec�ᱻExtensionLoader���м��ز����л��棬������忴һ��DubboCountCodec�ı���룻

```
private DubboCodec codec = new DubboCodec();
 
@Override
public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    codec.encode(channel, buffer, msg);
}
```
DubboCountCodec�ڲ����õ���DubboCodec��encode��������һ����ζ�Request������б���ģ������������£�

```
protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
       Serialization serialization = getSerialization(channel);
       // header.
       byte[] header = new byte[HEADER_LENGTH];
       // set magic number.
       Bytes.short2bytes(MAGIC, header);
 
       // set request and serialization flag.
       header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
 
       if (req.isTwoWay()) header[2] |= FLAG_TWOWAY;
       if (req.isEvent()) header[2] |= FLAG_EVENT;
 
       // set request id.
       Bytes.long2bytes(req.getId(), header, 4);
 
       // encode request data.
       int savedWriteIndex = buffer.writerIndex();
       buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
       ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
       ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
       if (req.isEvent()) {
           encodeEventData(channel, out, req.getData());
       } else {
           encodeRequestData(channel, out, req.getData(), req.getVersion());
       }
       out.flushBuffer();
       if (out instanceof Cleanable) {
           ((Cleanable) out).cleanup();
       }
       bos.flush();
       bos.close();
       int len = bos.writtenBytes();
       checkPayload(channel, len);
       Bytes.int2bytes(len, header, 12);
 
       // write
       buffer.writerIndex(savedWriteIndex);
       buffer.writeBytes(header); // write header.
       buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
   }
```
ǰ�����ֽڴ����ħ����0xdabb���������ֽڰ������ĸ���Ϣ�ֱ��ǣ��Ƿ���������Ϣ(������Ӧ��Ϣ)�����л����ͣ��Ƿ�˫��ͨ�ţ��Ƿ���������Ϣ��
��������Ϣ��ֱ�������˵��ĸ��ֽڣ�ֱ����5-12λ�ô����requestId����һ��long���ͣ����ĸ��ֽ�������Ǳ�����Ӧ��Ϣ�л�����Ӧ��״̬��
�������¿���buffer������HEADER_LENGTH���ȵ��ֽڣ������ʾ����header���ֵĳ���Ϊ16���ֽڣ�Ȼ��ͨ��ָ�������л���ʽ��data�������л���buffer�У����л�֮����Ի�ȡ��data�����ܹ����ֽ�������һ��int�����������ֽ�������int���ʹ����header������ĸ��ֽ��У�
����buffer��writerIndex���õ�д��header��data�ĵط�����ֹ���ݱ����ǣ�

###2.������
��NettyCodecAdapter�������ڲ���InternalEncoder��ͬ���ǵ���DubboCodec��decode���������ִ������£�

```
public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int readable = buffer.readableBytes();
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
        buffer.readBytes(header);
        return decode(channel, buffer, readable, header);
    }
 
    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        // check magic number.
        if (readable > 0 && header[0] != MAGIC_HIGH
                || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
            return super.decode(channel, buffer, readable, header);
        }
        // check length.
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }
 
        // get data length.
        int len = Bytes.bytes2int(header, 12);
        checkPayload(channel, len);
 
        int tt = len + HEADER_LENGTH;
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }
 
        // limit input stream.
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);
 
        try {
            return decodeBody(channel, is, header);
        } finally {
            if (is.available() > 0) {
                try {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Skip input stream " + is.available());
                    }
                    StreamUtils.skipUnusedStream(is);
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
    }
```
���ȶ�ȡMath.min(readable, HEADER_LENGTH)�����readableС��HEADER_LENGTH����ʾ���շ���ͷ����16���ֽڻ�û�����꣬��Ҫ�ȴ����գ�����header������֮����Ҫ���м�飬��Ҫ������ħ���ļ�飬header��Ϣ���ȼ�飬��Ϣ�峤�ȼ��(�����Ϣ���Ƿ��Ѿ��������)�������֮����Ҫ����Ϣ����з����л���������decodeBody�����У�

```
@Override
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
        // get request id.
        long id = Bytes.bytes2long(header, 4);
        if ((flag & FLAG_REQUEST) == 0) {
            // decode response.
            Response res = new Response(id);
            if ((flag & FLAG_EVENT) != 0) {
                res.setEvent(Response.HEARTBEAT_EVENT);
            }
            // get status.
            byte status = header[3];
            res.setStatus(status);
            if (status == Response.OK) {
                try {
                    Object data;
                    if (res.isHeartbeat()) {
                        data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                    } else if (res.isEvent()) {
                        data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                    } else {
                        DecodeableRpcResult result;
                        if (channel.getUrl().getParameter(
                                Constants.DECODE_IN_IO_THREAD_KEY,
                                Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                            result = new DecodeableRpcResult(channel, res, is,
                                    (Invocation) getRequestData(id), proto);
                            result.decode();
                        } else {
                            result = new DecodeableRpcResult(channel, res,
                                    new UnsafeByteArrayInputStream(readMessageData(is)),
                                    (Invocation) getRequestData(id), proto);
                        }
                        data = result;
                    }
                    res.setResult(data);
                } catch (Throwable t) {
                    if (log.isWarnEnabled()) {
                        log.warn("Decode response failed: " + t.getMessage(), t);
                    }
                    res.setStatus(Response.CLIENT_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
            } else {
                res.setErrorMessage(deserialize(s, channel.getUrl(), is).readUTF());
            }
            return res;
        } else {
            // decode request.
            Request req = new Request(id);
            req.setVersion(Version.getProtocolVersion());
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
            if ((flag & FLAG_EVENT) != 0) {
                req.setEvent(Request.HEARTBEAT_EVENT);
            }
            try {
                Object data;
                if (req.isHeartbeat()) {
                    data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                } else if (req.isEvent()) {
                    data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                } else {
                    DecodeableRpcInvocation inv;
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        inv = new DecodeableRpcInvocation(channel, req, is, proto);
                        inv.decode();
                    } else {
                        inv = new DecodeableRpcInvocation(channel, req,
                                new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                    }
                    data = inv;
                }
                req.setData(data);
            } catch (Throwable t) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode request failed: " + t.getMessage(), t);
                }
                // bad request
                req.setBroken(true);
                req.setData(t);
            }
            return req;
        }
    }
```
����ͨ������header���ֵĵ������ֽڣ�ʶ�����������Ϣ������Ӧ��Ϣ������ʹ���������͵����л���ʽ��Ȼ��ֱ�������л���

##���л��ͷ����л�
ͨ�����϶Ա��������������˽⣬�ڱ���������Ҫ���л�Request/Response���ڽ���������Ҫ���л�Request/Response��������忴�����л��ͷ����л���

###1.���л�
�ڱ���������Ҫ��ȡ�����Serialization������������£�

```
public static Serialization getSerialization(URL url) {
    return ExtensionLoader.getExtensionLoader(Serialization.class).getExtension(
            url.getParameter(Constants.SERIALIZATION_KEY, Constants.DEFAULT_REMOTING_SERIALIZATION));
}
```
ͬ��ȡcodec�ķ�ʽ��dubboҲ�ṩ�˶������л���ʽ��ͬʱ�����Զ�����չ��ͨ����url�л�ȡserialization�ؼ��֣������ȡ����Ĭ��Ϊhession2��ͬ���������л���Ҳ������һ���ļ��У�
·����META-INF/dubbo/internal/com.alibaba.dubbo.common.serialize.Serialization�������������£�

```
fastjson=com.alibaba.dubbo.common.serialize.fastjson.FastJsonSerialization
fst=com.alibaba.dubbo.common.serialize.fst.FstSerialization
hessian2=com.alibaba.dubbo.common.serialize.hessian2.Hessian2Serialization
java=com.alibaba.dubbo.common.serialize.java.JavaSerialization
compactedjava=com.alibaba.dubbo.common.serialize.java.CompactedJavaSerialization
nativejava=com.alibaba.dubbo.common.serialize.nativejava.NativeJavaSerialization
kryo=com.alibaba.dubbo.common.serialize.kryo.KryoSerialization
```
dubboĬ���ṩ��fastjson��fst��hessian2��java��compactedjava��nativejava��kryo�������л���ʽ��
ÿ�����л���ʽ����Ҫʵ�����������ӿ��ࣺSerialization��ObjectInput�Լ�ObjectOutput��
Serialization�ӿ��ࣺ

```
public interface Serialization {
 
    byte getContentTypeId();
 
    String getContentType();
 
    @Adaptive
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
 
    @Adaptive
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
 
}
```
���е�ContentTypeId������header�д�ŵ����л����ͣ������л���ʱ����Ҫͨ����id��ȡ�����Serialization�����Դ�ContentTypeId���ܳ����ظ��ģ�����ᱻ���ǣ�
ObjectInput�ӿ��ࣺ

```
public interface ObjectOutput extends DataOutput {
 
    void writeObject(Object obj) throws IOException;
}
```
ObjectOutput�ӿ��ࣺ

```
public interface ObjectInput extends DataInput {
 
    Object readObject() throws IOException, ClassNotFoundException;
 
    <T> T readObject(Class<T> cls) throws IOException, ClassNotFoundException;
 
    <T> T readObject(Class<T> cls, Type type) throws IOException, ClassNotFoundException;
}
```
�ֱ��ṩ�˶�ȡ�����д����Ľӿڷ�����DataOutput��DataInput�ֱ��ṩ�˶Ի����������͵Ķ���д�����л�ֻ��Ҫ����writeObject������Dataд�����������ɣ�������Կ�һ�±������е��õ�encodeRequestData������

```
@Override
protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
    RpcInvocation inv = (RpcInvocation) data;
 
    out.writeUTF(version);
    out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
    out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));
 
    out.writeUTF(inv.getMethodName());
    out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
    Object[] args = inv.getArguments();
    if (args != null)
        for (int i = 0; i < args.length; i++) {
            out.writeObject(encodeInvocationArgument(channel, inv, i));
        }
    out.writeObject(inv.getAttachments());
}
```
Ĭ��ʹ�õ�DubboCountCodec��ʽ��û��ֱ�ӽ�dataд�����У����ǽ�RpcInvocation�е�����ȡ���ֱ�д������

###2.�����л�
�����л�ͨ����ȡheader�е����л����ͣ�Ȼ��ͨ�����·�����ȡ�����Serialization����������CodecSupport�У�

```
public static Serialization getSerialization(URL url, Byte id) throws IOException {
    Serialization serialization = getSerializationById(id);
    String serializationName = url.getParameter(Constants.SERIALIZATION_KEY, Constants.DEFAULT_REMOTING_SERIALIZATION);
    // Check if "serialization id" passed from network matches the id on this side(only take effect for JDK serialization), for security purpose.
    if (serialization == null
            || ((id == 3 || id == 7 || id == 4) && !(serializationName.equals(ID_SERIALIZATIONNAME_MAP.get(id))))) {
        throw new IOException("Unexpected serialization id:" + id + " received from network, please check if the peer send the right id.");
    }
    return serialization;
}
 
private static Map<Byte, Serialization> ID_SERIALIZATION_MAP = new HashMap<Byte, Serialization>();
 
public static Serialization getSerializationById(Byte id) {
    return ID_SERIALIZATION_MAP.get(id);
}
```
ID_SERIALIZATION_MAP�����ContentTypeId�;���Serialization�Ķ�Ӧ��ϵ��Ȼ��ͨ��id��ȡ�����Serialization��Ȼ�����д���˳���ȡ���ݣ�

##��չ���л�����
dubbo����Ժܶ�ģ���ṩ�˺ܺõ���չ���ܣ��������л����ܣ�����������һ�����ʹ��protobuf��ʵ�����л���ʽ��

###1.�������ṹ
���ȿ�һ������Ĵ���ṹ������ͼ��ʾ��
![ͼƬ����][2]
�ֱ�ʵ�������ӿ��ࣺSerialization��ObjectInput�Լ�ObjectOutput��Ȼ����ָ��Ŀ¼���ṩһ���ı��ļ���

###2.������չ��

```
<dependency>
     <groupId>com.dyuproject.protostuff</groupId>
     <artifactId>protostuff-core</artifactId>
     <version>1.1.3</version>
</dependency>
<dependency>
     <groupId>com.dyuproject.protostuff</groupId>
     <artifactId>protostuff-runtime</artifactId>
     <version>1.1.3</version>
</dependency>
```
###3.ʵ�ֽӿ�ObjectInput��ObjectOutput

```
public class ProtobufObjectInput implements ObjectInput {
 
    private ObjectInputStream input;
 
    public ProtobufObjectInput(InputStream inputStream) throws IOException {
        this.input = new ObjectInputStream(inputStream);
    }
 
    ....ʡ�Ի�������...
     
    @Override
    public Object readObject() throws IOException, ClassNotFoundException {
        return input.readObject();
    }
 
    @Override
    public <T> T readObject(Class<T> clazz) throws IOException {
        try {
            byte[] buffer = (byte[]) input.readObject();
            input.read(buffer);
            return SerializationUtil.deserialize(buffer, clazz);
        } catch (Exception e) {
            throw new IOException(e);
        }
 
    }
 
    @Override
    public <T> T readObject(Class<T> clazz, Type type) throws IOException {
        try {
            byte[] buffer = (byte[]) input.readObject();
            input.read(buffer);
            return SerializationUtil.deserialize(buffer, clazz);
        } catch (Exception e) {
            throw new IOException(e);
        }
    }
}
 
public class ProtobufObjectOutput implements ObjectOutput {
 
    private ObjectOutputStream outputStream;
 
    public ProtobufObjectOutput(OutputStream outputStream) throws IOException {
        this.outputStream = new ObjectOutputStream(outputStream);
    }
 
    ....ʡ�Ի�������...
 
    @Override
    public void writeObject(Object v) throws IOException {
        byte[] bytes = SerializationUtil.serialize(v);
        outputStream.writeObject(bytes);
        outputStream.flush();
    }
 
    @Override
    public void flushBuffer() throws IOException {
        outputStream.flush();
    }
}
```
4.ʵ��Serialization�ӿ�

```
public class ProtobufSerialization implements Serialization {
 
    @Override
    public byte getContentTypeId() {
        return 10;
    }
 
    @Override
    public String getContentType() {
        return "x-application/protobuf";
    }
 
    @Override
    public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
        return new ProtobufObjectOutput(out);
    }
 
    @Override
    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        return new ProtobufObjectInput(is);
    }
}
```
����������һ���µ�ContentTypeId����Ҫ��֤��dubbo�����Ѵ��ڵĲ�Ҫ��ͻ

###5.ָ��Ŀ¼�ṩע��
��META-INF/dubbo/internal/Ŀ¼���ṩ�ļ�com.alibaba.dubbo.common.serialize.Serialization���������£�

```
protobuf=com.dubboCommon.ProtobufSerialization
```
###6.���ṩ�������µ����л���ʽ

```
<dubbo:protocol name="dubbo" port="20880" serialization="protobuf"/>
```
�����ͻ�ʹ������չ��protobuf���л���ʽ�����л�����

##�ܽ�
���Ĵ�dubbo������Ƶ���ײ�serialization�����������˽�dubbo������������з�������dubbo��һ������͸�����˽⣻

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][3]


  [1]: /img/bVbbUbQ
  [2]: /img/bVbhTZR
  [3]: https://github.com/ksfzhaohui/blog