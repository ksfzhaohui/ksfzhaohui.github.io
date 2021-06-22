## 概念
​首先解释下内存溢出和内存泄露的概念。内存溢出一般指的是out of memory，也就是我们经常说的OOM，常发生在堆，方法区和方法栈。内存泄露指的是一段程序在申请内存空间后，无法释放已经申请的内存空间，导致该内存地址不可达，后续程序里这块内存空间永远被占用。就好像商场的物品柜设计了10个抽屉，每个人使用后都会归还给下一个用户使用，如果有某个人一直占用不退还，别的用户就只能使用剩下的9个抽屉，这样的人多了以后，最后大家一个抽屉也无法使用了。所以内存泄露跟内存溢出是存在联系的，一次内存泄露不会有太大的影响，内存泄露堆积以后就会导致内存溢出。

此处应该有图，大家脑补一下画面。

## 需求背景
一个to C的个人账户系统，数据量大概有2000多W；某天产品突然来说要做一个to B 商户账户系统，马上一个星期左右后的大促活动就要用，产品跟大老板牛逼已经吹出去了，说我们的系统已经具备了这个能力，能够立马无缝支持。现在火急火燎的来找我们，问是不是直接把我们这套to C的账户系统的能力提供出去就可以了。

## 需求分析
首先在这么短的时间内要开发测试上线，新做一个to B的账户系统肯定是不现实的，咱们程序员能力再强也不能流水线生产系统啊。只能依赖当前账户系统的能力先去支撑这个业务。这两套账户体系底层的核心领域模型可以抽象一致，都包括记账凭证，记账主体，记账流水。不同的地方在于不同的业务使用场景下进出帐的规则不一样，所以规则层的领域模型需要定制化的配置开发。同时目前系统已经有2000多W账户，日增加流水也是10W级别的，本身数据量已经很大，而且业务上两套数据会相互影响，需要在业务上做垂直分表，将两套数据隔离。

## 实施过程
领域模型和数据存储方案定下来后，马上就开始实施了，当时为了更优雅的实现，对目前to C的业务影响最小，我们特意在interface入口处统一拦截后在当前工作线程的threadlocal加上一个判断标记，用来识别是to C的业务请求还是to B的业务请求，然后业务规则层通过不同的filter进行过滤，最后在数据库jdbc层通过不同的标记对不同的表做相应的DML处理。方案实施的很快，两三天就开发完提测，可在测试过程中发现了两个问题，第一个问题是数据有时候会无缘无故的错乱，本来是写到to B表结构的数据会写到了to C表结构里。而且压测时发现内存监控曲线在慢慢的增长，发生了内存泄露。

## 问题分析
仔细分析源代码后发现问题出现在threadlocal。每个线程中都有一个ThreadLocalMap数据结构，当执行set方法时，其值是保存在当前线程的threadLocals变量中，当执行get方法中，是从当前线程的threadLocals变量获取。所以在线程1中set的值，对线程2来说是摸不到的，而且在线程2中重新set的话，也不会影响到线程1中的值，保证了线程之间不会相互干扰
```
public void set(T value) {
   Thread t = Thread.currentThread();
   ThreadLocalMap map = getMap(t);
   if (map != null)
       map.set(this, value);
   else
       createMap(t, value);
}

public T get() {
   Thread t = Thread.currentThread();
   ThreadLocalMap map = getMap(t);
   if (map != null) {
       ThreadLocalMap.Entry e = map.getEntry(this);
       if (e != null) {
           @SuppressWarnings("unchecked")
           T result = (T)e.value;
           return result;
      }
  }
   return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
   return t.threadLocals;
}

//Entry为ThreadLocalMap静态内部类，对ThreadLocal的若引用
//同时让ThreadLocal和储值形成key-value的关系
static class Entry extends WeakReference<ThreadLocal<?>> {
   /** The value associated with this ThreadLocal. */
   Object value;

   Entry(ThreadLocal<?> k, Object v) {
          super(k);
           value = v;
  }
}
```


## 内存泄露分析
通过代码可以知道，当一个线程调用ThreadLocal的set方法设置变量时候，当前线程的ThreadLocalMap里面就会存放一个Entry对象，这个记录的key为ThreadLocal的引用，value则为设置的值。如果当前线程一直存在而没有调用ThreadLocal的remove方法，并且这时候其它地方还是有对ThreadLocal的引用，则当前线程的ThreadLocalMap变量里面会存在ThreadLocal变量的引用和value对象的引用是不会被释放的，这就会造成内存泄露的。但是考虑如果这个ThreadLocal变量没有了其他强依赖，而当前线程还存在的情况下，由于线程的ThreadLocalMap里面的key是弱依赖，则当前线程的ThreadLocalMap里面的ThreadLocal变量的弱引用会被在gc的时候回收，但是对应value还是会造成内存泄露。

## 数据写错分析
java进程是通过外部web容器如tomcat启动的，web容器启动时会启动一个线程池，创建一些核心初始线程，这些线程处理完一个任务后会继续处理另外一个任务。这个时候假如核心线程刚处理完一个to B的任务，处理完后线程没有消失继续处理一个to C的任务，这个时候threadLocal里的变量标记还是直接的标记，就会导致在jdbc层继续被路由到to B的表结构里，导致数据错误。

## 修复处理
了解清楚threadLoca的存储结构后，在interface入口处做一个AOP切片，通过环绕通知，在方法正式处理前在当前工作线程的threadlocal set一个判断标记，方法处理完成后手动remve掉这个标记，保证每次处理后 内存正确被释放。

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。