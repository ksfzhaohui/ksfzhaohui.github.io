**Netty流量统计**  
netty专门提供了一个traffic包用于流量的统计，如下图所示：

![](https://static.oschina.net/uploads/space/2016/1119/154414_NUEo_159239.png)

分别提供了全局的GlobalTrafficShapingHandler和针对channel的ChannelTrafficShapingHandler，同时提供了TrafficCounter用来记录实时的流量统计。

简单的使用：

```
ChannelPipeline p = socketChannel.pipeline();
p.addLast(new GlobalTrafficShapingHandler(Executors.newScheduledThreadPool(1), 1000));
```

GlobalTrafficShapingHandler提供的方法trafficCounter()可以用来获取TrafficCounter对象，  
TrafficCounter提供了常用的一些方法：

```
/**
     * @return the Read Throughput in bytes/s computes in the last check interval.
     */
    public long lastReadThroughput() {
        return lastReadThroughput;
    }

    /**
     * @return the Write Throughput in bytes/s computes in the last check interval.
     */
    public long lastWriteThroughput() {
        return lastWriteThroughput;
    }

    /**
     * @return the number of bytes read during the last check Interval.
     */
    public long lastReadBytes() {
        return lastReadBytes;
    }

    /**
     * @return the number of bytes written during the last check Interval.
     */
    public long lastWrittenBytes() {
        return lastWrittenBytes;
    }

    /**
     * @return the current number of bytes read since the last checkInterval.
     */
    public long currentReadBytes() {
        return currentReadBytes.get();
    }

    /**
     * @return the current number of bytes written since the last check Interval.
     */
    public long currentWrittenBytes() {
        return currentWrittenBytes.get();
    }

    /**
     * @return the Time in millisecond of the last check as of System.currentTimeMillis().
     */
    public long lastTime() {
        return lastTime.get();
    }

    /**
     * @return the cumulativeWrittenBytes
     */
    public long cumulativeWrittenBytes() {
        return cumulativeWrittenBytes.get();
    }

    /**
     * @return the cumulativeReadBytes
     */
    public long cumulativeReadBytes() {
        return cumulativeReadBytes.get();
    }

    /**
     * @return the lastCumulativeTime in millisecond as of System.currentTimeMillis()
     * when the cumulative counters were reset to 0.
     */
    public long lastCumulativeTime() {
        return lastCumulativeTime;
    }

    /**
     * @return the realWrittenBytes
     */
    public AtomicLong getRealWrittenBytes() {
        return realWrittenBytes;
    }

    /**
     * @return the realWriteThroughput
     */
    public long getRealWriteThroughput() {
        return realWriteThroughput;
    }
```

**JMX的MBean**

JMX即Java Management Extensions(Java管理扩展),MBean即Managed Beans(被管理的Beans)  
一个MBean是一个被管理的Java对象，有点类似于JavaBean，一个设备、一个应用或者任何资源都可以被表示为MBean，MBean会暴露一个接口对外，这个接口可以读取或者写入一些对象中的属性，通常一个MBean需要定义一个接口，以MBean结尾，如下面用于统计netty的流量的MBean：

```
public interface IoAcceptorStatMBean {

    public long getWrittenBytesThroughput();
    
    public long getReadBytesThroughput();
}
```

MBean的实现：

```
public class IoAcceptorStat implements IoAcceptorStatMBean {

    @Override
    public long getWrittenBytesThroughput() {
        return SocksServer.getInstance().getTrafficCounter()
                .lastWriteThroughput();
    }

    @Override
    public long getReadBytesThroughput() {
        return SocksServer.getInstance().getTrafficCounter()
                .lastReadThroughput();
    }

}
```

下面需要将我们定义的MBean被管理起来

```
MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
IoAcceptorStat mbean = new IoAcceptorStat();

try {
     ObjectName acceptorName = new ObjectName(mbean.getClass().getPackage().getName()
                 + ":type=IoAcceptorStat");
     mBeanServer.registerMBean(mbean, acceptorName);
} catch (Exception e) {
     e.printStackTrace();
}
```

1.向ManagementFactory申请了一个MBeanServer对象  
2.给定一个标识ObjectName，使用了(包名:type=类名)的形式  
3.通过registerMBean方法注册了mbean，并且使用ObjectName作为标识

接下来就可以使用JDK自带工具jconsole来查看MBean了

**监控netty的流量**

以上的代码是项目shadowsocks-netty中的部分代码，shadowsocks-netty是一个基于netty实现的shadowsocks的客户端，  
更多介绍：[https://my.oschina.net/OutOfMemory/blog/744475](https://my.oschina.net/OutOfMemory/blog/744475)  
github：[https://github.com/ksfzhaohui/shadowsocks-netty](https://github.com/ksfzhaohui/shadowsocks-netty)

启动shadowsocks-netty程序，使用jconsole进行连接，并打开YouTube随便打开一个视频，切换到Jconsole的MBean页签就可以实时监控了

![](https://static.oschina.net/uploads/space/2016/1119/154732_NYhk_159239.png)

监控的代理程序的写速度，其实就是浏览器下载数据的速度

![](https://static.oschina.net/uploads/space/2016/1119/154756_w2XG_159239.png)

监控的代理程序的读速度，其实就是浏览器的上传速度

**总结**

在开发阶段对应用流量的监控还是很有必要的，可以大概估算应用的带宽，同时方便我们查看是否达到指定的标准

**个人博客：[codingo.xyz](http://codingo.xyz/)**