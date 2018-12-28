**Disruptor和LinkedBlockingQueue简介**  
Disruptor是Java实现的用于线程间通信的消息组件，其核心是一个Lock-free(无锁)的Ringbuffer；LinkedBlockingQueue是java.util.concurrent包中提供的一个阻塞队列；因为二者之间有很多相同的地方，所以在此进行一次性能的对比。

**压力测试**  
1.针对LinkedBlockingQueue的压测类

```
public class LinkedBlockingQueueTest {

    public static int eventNum = 5000000;

    public static void main(String[] args) {
        final BlockingQueue<LogEvent> queue = new LinkedBlockingQueue<LogEvent>();
        final long startTime = System.currentTimeMillis();
        new Thread(new Runnable() {

            @Override
            public void run() {
                int i = 0;
                while (i < eventNum) {
                    LogEvent logEvent = new LogEvent(i, "c" + i);
                    try {
                        queue.put(logEvent);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    i++;
                }
            }
        }).start();

        new Thread(new Runnable() {

            @Override
            public void run() {
                int k = 0;
                while (k < eventNum) {
                    try {
                        queue.take();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    k++;
                }
                long endTime = System.currentTimeMillis();
                System.out
                        .println("costTime = " + (endTime - startTime) + "ms");
            }
        }).start();
    }
}
```

LinkedBlockingQueueTest 实现了一个简单的生产者-消费者模式，一条线程负责插入，另外一条线程负责读取。

```
public class LogEvent implements Serializable {

    private static final long serialVersionUID = 1L;
    private long logId;
    private String content;
    
    public LogEvent(){
        
    }
    
    public LogEvent(long logId, String content){
        this.logId = logId;
        this.content = content;
    }

    public long getLogId() {
        return logId;
    }

    public void setLogId(long logId) {
        this.logId = logId;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

LogEvent实体类，Disruptor的压测类中也同样会用到

2.下面是针对Disruptor的压测类，需要引入Disruptor的jar包

```
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.3.6</version>
</dependency>
```

Disruptor的压测类

```
public class DisruptorTest {

    public static void main(String[] args) {
        LogEventFactory factory = new LogEventFactory();
        int ringBufferSize = 65536;
        final Disruptor<LogEvent> disruptor = new Disruptor<LogEvent>(factory,
                ringBufferSize, DaemonThreadFactory.INSTANCE,
                ProducerType.SINGLE, new BusySpinWaitStrategy());

        LogEventConsumer consumer = new LogEventConsumer();
        disruptor.handleEventsWith(consumer);
        disruptor.start();
        new Thread(new Runnable() {

            @Override
            public void run() {
                RingBuffer<LogEvent> ringBuffer = disruptor.getRingBuffer();
                for (int i = 0; i < LinkedBlockingQueueTest.eventNum; i++) {
                    long seq = ringBuffer.next();
                    LogEvent logEvent = ringBuffer.get(seq);
                    logEvent.setLogId(i);
                    logEvent.setContent("c" + i);
                    ringBuffer.publish(seq);
                }
            }
        }).start();
    }
}
```

同样为了保证测试数据的准确性，Disruptor使用了ProducerType.SINGLE(单生产者)模式，同时也只使用了一个LogEventConsumer(消费者)

```
public class LogEventConsumer implements EventHandler<LogEvent> {

    private long startTime;
    private int i;

    public LogEventConsumer() {
        this.startTime = System.currentTimeMillis();
    }

    public void onEvent(LogEvent logEvent, long seq, boolean bool)
            throws Exception {
        i++;
        if (i == LinkedBlockingQueueTest.eventNum) {
            long endTime = System.currentTimeMillis();
            System.out.println(" costTime = " + (endTime - startTime) + "ms");
        }
    }

}
```

LogEventConsumer 中负责记录开始时间和结束时间以及接受消息的数量，方便统计时间

**压测结果统计**  
测试环境：  
操作系统：win7 32位  
CPU：Intel Core i3-2350M 2.3GHz 4核  
内存：3G  
JDK：1.6

分别运行以上两个实例，运行多次取平均值，结果如下：

![](https://static.oschina.net/uploads/space/2016/1123/233322_GWS6_159239.png)

结果显示Disruptor是LinkedBlockingQueue的1.65倍，测试环境是本人的笔记本电脑，配置有点低所有差距并不是特别明显；同样在公司台式机（win7 64位 – Intel Core i5 4核 – 4g内存 – jdk1.7）显示的结果是3-4倍左右；官方提供的数据是在5倍左右：[https://github.com/LMAX-Exchange/disruptor/wiki/Performance-Results](https://github.com/LMAX-Exchange/disruptor/wiki/Performance-Results)

**性能差距原因分析**  
1.lock和cas的差距  
LinkedBlockingQueue中使用了锁，如下所示：

```
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();
```

而Disruptor中提供了cas的无锁支持，提供了BusySpinWaitStrategy策略的支持

2.避免伪共享  
缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

看一个实例：

```
public class FalseSharing implements Runnable {
    public final static int NUM_THREADS = 4;
    public final static long ITERATIONS = 50000000;
    private final int arrayIndex;

    private static VolatileLong[] longs = new VolatileLong[NUM_THREADS];
    static {
        for (int i = 0; i < longs.length; i++) {
            longs[i] = new VolatileLong();
        }
    }

    public FalseSharing(final int arrayIndex) {
        this.arrayIndex = arrayIndex;
    }

    public static void main(final String[] args) throws Exception {
        final long start = System.currentTimeMillis();
        runTest();
        System.out.println("costTime = " + (System.currentTimeMillis() - start) + "ms");
    }

    private static void runTest() throws InterruptedException {
        Thread[] threads = new Thread[NUM_THREADS];

        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new FalseSharing(i));
        }

        for (Thread t : threads) {
            t.start();
        }

        for (Thread t : threads) {
            t.join();
        }
    }

    @Override
    public void run() {
        long i = ITERATIONS + 1;
        while (0 != --i) {
            longs[arrayIndex].value = i;
        }
    }

    public final static class VolatileLong {
        public volatile long value = 0L;
        public long p1, p2, p3, p4, p5, p6;
    }
}
```

分别注释掉VolatileLong 中的public long p1, p2, p3, p4, p5, p6;和不注释掉进行对比，发现不注释掉的性能居然是注释掉性能的4倍，原因就是缓存行大小是64个字节，不注释掉说明一个VolatileLong 对象刚好占用一个缓存行；注释掉的话一个缓存行会被多个变量占用，就会无意中影响彼此的性能。

查看Disruptor的源码会发现很多地方避免了伪共享，比如：

```
abstract class SingleProducerSequencerPad extends AbstractSequencer
{
    protected long p1, p2, p3, p4, p5, p6, p7;

    public SingleProducerSequencerPad(int bufferSize, WaitStrategy waitStrategy)
    {
        super(bufferSize, waitStrategy);
    }
}
```

3.Ringbuffer的使用  
Disruptor选择使用Ringbuffer来构造lock-free队列，什么事Ringbuffer，可以参考wiki：[https://zh.wikipedia.org/wiki/%E7%92%B0%E5%BD%A2%E7%B7%A9%E8%A1%9D%E5%8D%80](https://zh.wikipedia.org/wiki/%E7%92%B0%E5%BD%A2%E7%B7%A9%E8%A1%9D%E5%8D%80)  
数组是预分配的，这样避免了Java GC带来的运行开销。生产者在生产消息或产生事件的时候对Ringbuffer元素中的属性进行更新，而不是替换Ringbuffer中的元素。

占时先整理这三条，肯定还有其他原因

**总结**  
Disruptor的高性能早就被用在了一些第三方库中，比如log4j2，让log4j2在性能上有质的飞越，之前对三种主流日志性能对比：[https://my.oschina.net/OutOfMemory/blog/789267](https://my.oschina.net/OutOfMemory/blog/789267)