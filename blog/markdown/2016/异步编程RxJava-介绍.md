**前言**  
前段时间写了一篇[对协程的一些理解](https://my.oschina.net/OutOfMemory/blog/803349)，里面提到了不管是协程还是callback，本质上其实提供的是一种异步无阻塞的编程模式；并且介绍了java中对异步无阻赛这种编程模式的支持，主要提到了Future和CompletableFuture；之后有同学在下面留言提到了RxJava，刚好最近在看[微服务设计](https://book.douban.com/subject/26772677/)这本书，里面提到了响应式扩展(Reactive extensions,Rx)，而RxJava是Rx在JVM上的实现，所有打算对RxJava进一步了解。

**RxJava简介**  
RxJava的官网地址：[https://github.com/ReactiveX/RxJava](https://github.com/ReactiveX/RxJava)，  
其中对RxJava进行了一句话描述：RxJava – Reactive Extensions for the JVM – a library for composing asynchronous and event-based programs using observable sequences for the Java VM.  
大意就是：一个在Java VM上使用可观测的序列来组成异步的、基于事件的程序的库。  
更详细的说明在Netflix技术博客的一篇文章中描述了RxJava的主要特点：  
1.易于并发从而更好的利用服务器的能力。  
2.易于有条件的异步执行。  
3.一种更好的方式来避免回调地狱。  
4.一种响应式方法。

**与CompletableFuture对比**  
之前提到CompletableFuture真正的实现了异步的编程模式，一个比较常见的使用场景：

```
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(耗时函数);
Future<Integer> f = future.whenComplete((v, e) -> {
        System.out.println(v);
        System.out.println(e);
});
System.out.println("other...");
```

下面用一个简单的例子来看一下RxJava是如何实现异步的编程模式：

```
Observable<Long> observable = Observable.just(1, 2)
        .subscribeOn(Schedulers.io()).map(new Func1<Integer, Long>() {
            @Override
            public Long call(Integer t) {
                try {
                    Thread.sleep(1000); //耗时的操作
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return (long) (t * 2);
            }
        });
observable.subscribe(new Subscriber<Long>() {
    @Override
    public void onCompleted() {
        System.out.println("onCompleted");
    }
    @Override
    public void onError(Throwable e) {
        System.out.println("error" + e);
    }
    @Override
    public void onNext(Long result) {
        System.out.println("result = " + result);
    }
});
System.out.println("other...");
```

Func1中以异步的方式执行了一个耗时的操作，Subscriber(观察者)被订阅到Observable(被观察者)中，当耗时操作执行完会回调Subscriber中的onNext方法。  
其中的异步方式是在subscribeOn(Schedulers.io())中指定的，Schedulers.io()可以理解为每次执行耗时操作都启动一个新的线程。  
结构上其实和CompletableFuture很像，都是异步的执行一个耗时的操作，然后在有结果的时候主动告诉我结果。那我们还需要RxJava干嘛，不知道你有没有注意，上面的例子中其实提供2条数据流\[1,2\]，并且处理完任何一个都会主动告诉我，当然这只是它其中的一项功能，RxJava还有很多好用的功能，在下面的内容会进行介绍。

**异步观察者模式**  
上面这段代码有没有发现特别像设计模式中的：观察者模式；首先提供一个被观察者Observable，然后把观察者Subscriber添加到了被观察者列表中；  
RxJava中一共提供了四种角色：Observable、Observer、Subscriber、Subjects  
Observables和Subjects是两个被观察者，Observers和Subscribers是观察者；  
当然我们也可以查看一下源码，看一下jdk中的Observer和RxJava的Observer  
jdk中的Observer：

```
public interface Observer {
    void update(Observable o, Object arg);
}
```

RxJava的Observer:

```
public interface Observer<T> {
    void onCompleted();
    void onError(Throwable e);
    void onNext(T t);
}
```

同时可以发现Subscriber是implements Observer的：

```
public abstract class Subscriber<T> implements Observer<T>, Subscription
```

可以发现RxJava中在Observer中引入了2个新的方法：onCompleted()和onError()  
onCompleted()：即通知观察者Observable没有更多的数据，事件队列完结  
onError():在事件处理过程中出异常时，onError()会被触发,同时队列自动终止，不允许再有事件发出。  
正是因为RxJava提供了同步和异步两种方式进行事件的处理，个人觉得异步的方式更能体现RxJava的价值，所以这里给他命名为**异步观察者模式**。

好了，下面正式介绍RxJava的那些灵活的操作符，这里仅仅是简单的介绍和简单的实例，具体用在什么场景下，会在以后的文章中介绍

**Maven引入**

```
<dependency>
    <groupId>io.reactivex</groupId>
    <artifactId>rxjava</artifactId>
    <version>1.2.4</version>
</dependency>
```

**创建Observable**  
1.create()创建一个Observable，并为它定义事件触发规则

```
Observable<Integer> observable = Observable
            .create(new Observable.OnSubscribe<Integer>() {
                @Override
                public void call(Subscriber<? super Integer> observer) {
                    for (int i = 0; i < 5; i++) {
                        observer.onNext(i);
                    }
                    observer.onCompleted();
                }
            });
observable.subscribe(new Observer<Integer>() {...});
```

2.from()可以从一个列表中创建一个Observable，Observable将发射出列表中的每一个元素

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items);
observable.subscribe(new Observer<Integer>() {...});
```

3.just()将传入的参数依次发送出来

```
Observable<Integer> observable = Observable.just(1, 2, 3);
observable.subscribe(new Observer<Integer>() {...});
```

**过滤Observable**  
1.filter()来过滤我们观测序列中不想要的值

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).filter(
        new Func1<Integer, Boolean>() {
            @Override
            public Boolean call(Integer t) {
                return t == 1;
            }
        });
observable.subscribe(new Observer<Integer>() {...});
```

2.take()和taskLast()分别取前几个元素和后几个元素

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).take(3);
observable.subscribe(new Observer<Integer>() {...});
```

```
Observable<Integer> observable = Observable.from(items).takeLast(2);
```

3.distinct()和distinctUntilChanged()  
distinct()过滤掉重复的值

```
List<Integer> items = new ArrayList<Integer>();
items.add(1);
items.add(10);
items.add(10);
Observable<Integer> observable = Observable.from(items).distinct();
observable.subscribe(new Observer<Integer>() {...});
```

distinctUntilChanged()列发射一个不同于之前的一个新值时让我们得到通知

```
List<Integer> items = new ArrayList<Integer>();
items.add(1);
items.add(100);
items.add(100);
items.add(200);
Observable<Integer> observable = Observable.from(items).distinctUntilChanged();
observable.subscribe(new Observer<Integer>() {...});
```

4.first()和last()分别取第一个元素和最后一个元素

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
// Observable<Integer> observable = Observable.from(items).first();
Observable<Integer> observable = Observable.from(items).last();
observable.subscribe(new Observer<Integer>() {...});
```

5.skip()和skipLast()分别从前或者后跳过几个元素

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
// Observable<Integer> observable = Observable.from(items).skip(2);
Observable<Integer> observable = Observable.from(items).skipLast(2);
observable.subscribe(new Observer<Integer>() {...});
```

6.elementAt()取第几个元素进行发射

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).elementAt(2);
observable.subscribe(new Observer<Integer>() {...});
```

7.sample()指定发射间隔进行发射

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 50000; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).sample(1,TimeUnit.MICROSECONDS);
observable.subscribe(new Observer<Integer>() {...});
```

8.timeout()设定的时间间隔内如果没有得到一个值则发射一个错误

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).timeout(1,TimeUnit.MICROSECONDS);
observable.subscribe(new Observer<Integer>() {...onError()...});

```

9.debounce()在一个指定的时间间隔过去了仍旧没有发射一个，那么它将发射最后的那个

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).debounce(1,TimeUnit.MICROSECONDS);
observable.subscribe(new Observer<Integer>() {...});
```

**转换Observable**  
1.map()接收一个指定的Func对象然后将它应用到每一个由Observable发射的值上

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).map(
        new Func1<Integer, Integer>() {
            @Override
            public Integer call(Integer t) {
                return t * 2;
            }
        });
observable.subscribe(new Observer<Integer>() {...});
```

2.flatMap()函数提供一种铺平序列的方式，然后合并这些Observables发射的数据

```
final Scheduler scheduler = Schedulers.from(Executors.newFixedThreadPool(3));
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).flatMap(
        new Func1<Integer, Observable<? extends Integer>>() {
            @Override
            public Observable<? extends Integer> call(Integer t) {
                List<Integer> items = new ArrayList<Integer>();
                items.add(t);
                items.add(99999);
                return Observable.from(items).subscribeOn(scheduler);
            }
        });
observable.subscribe(new Observer<Integer>() {...});
```

重要的一点提示是关于合并部分：它允许交叉。这意味着flatMap()不能够保证在最终生成的Observable中源Observables确切的发射  
顺序。

3.concatMap()函数解决了flatMap()的交叉问题，提供了一种能够把发射的值连续在一起的铺平函数，而不是合并它们。  
示例代码同上，将flatMap替换为concatMap，输出的结果来看是有序的

4.switchMap()和flatMap()很像，除了一点：每当源Observable发射一个新的数据项（Observable）时，它将取消订阅并停止监视之前那个数据项产生的Observable，并开始监视当前发射的这一个。  
示例代码同上，将flatMap替换为switchMap，输出的结果只剩最后一个值

5.scan()是一个累积函数，对原始Observable发射的每一项数据都应用一个函数，计算出函数的结果值，并将该值填充回可观测序列，等待和下一次发射的数据一起使用。

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Integer> observable = Observable.from(items).scan(
        new Func2<Integer, Integer, Integer>() {
 
            @Override
            public Integer call(Integer t1, Integer t2) {
                System.out.println(t1 + "+" + t2);
                return t1 + t2;
            }
        });
observable.subscribe(new Observer<Integer>() {...});
```

6.groupBy()来分组元素

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<GroupedObservable<Integer, Integer>> observable = Observable
                .from(items).groupBy(new Func1<Integer, Integer>() {
                    @Override
                    public Integer call(Integer t) {
                        return t % 3;
                    }
                });
observable.subscribe(new Observer<GroupedObservable<Integer, Integer>>() {
        @Override
        public void onNext(final GroupedObservable<Integer, Integer> t) {
            t.subscribe(new Action1<Integer>() {
                @Override
                public void call(Integer value) {
                    System.out.println("key:" + t.getKey()+ ", value:" + value);
                }
            });
                  
});
```

7.buffer()函数将源Observable变换一个新的Observable，这个新的Observable每次发射一组列表值而不是一个一个发射。

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<List<Integer>> observable = Observable.from(items).buffer(2);
observable.subscribe(new Observer<List<Integer>>() {...});
```

8.window()函数和 buffer()很像，但是它发射的是Observable而不是列表

```
List<Integer> items = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items.add(i);
}
Observable<Observable<Integer>> observable = Observable.from(items).window(2);
observable.subscribe(new Observer<Observable<Integer>>() {
    @Override
    public void onNext(Observable<Integer> t) {
        t.subscribe(new Action1<Integer>() {
            @Override
            public void call(Integer t) {
                System.out.println("this Action1 = " + this+ ",result = " + t);
            }
        });
        //onCompleted和onError
});
```

9.cast()它将源Observable中的每一项数据都转换为新的类型，把它变成了不同的Class

```
List<Father> items = new ArrayList<Father>();
items.add(new Son());
items.add(new Son());
items.add(new Father());
items.add(new Father());
Observable<Son> observable = Observable.from(items).cast(Son.class);
observable.subscribe(new Observer<Son>() {...});
 
class Father {
}
 
class Son extends Father {
}
```

**组合Observables**  
1.merge()方法将帮助你把两个甚至更多的Observables合并到他们发射的数据项里

```
List<Integer> items1 = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items1.add(i);
}
List<Integer> items2 = new ArrayList<Integer>();
for (int i = 5; i < 10; i++) {
    items2.add(i);
}
Observable<Integer> observable1 = Observable.from(items1);
Observable<Integer> observable2 = Observable.from(items2);
Observable<Integer> observableMerge = Observable.merge(observable1,observable2);
observable.subscribe(new Observer<Integer>() {...});
```

2.zip()合并两个或者多个Observables发射出的数据项，根据指定的函数 Func* 变换它们，并发射一个新值

```
List<Integer> items1 = new ArrayList<Integer>();
for (int i = 0; i < 5; i++) {
    items1.add(i);
}
List<Integer> items2 = new ArrayList<Integer>();
for (int i = 5; i < 10; i++) {
    items2.add(i);
}
Observable<Integer> observable1 = Observable.from(items1);
Observable<Integer> observable2 = Observable.from(items2);
Observable<Integer> observableZip = Observable.zip(observable1,
        observable2, new Func2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer t1, Integer t2) {
                return t1 * t2;
            }
        });
observable.subscribe(new Observer<Integer>() {...});
```

3.combineLatest()把两个Observable产生的结果进行合并，这两个Observable中任意一个Observable产生的结果，都和另一个Observable最后产生的结果，按照一定的规则进行合并。

```
Observable<Long> observable1 = Observable.interval(1000,TimeUnit.MILLISECONDS);
Observable<Long> observable2 = Observable.interval(1000,TimeUnit.MILLISECONDS);
Observable.combineLatest(observable1, observable2,
        new Func2<Long, Long, Long>() {
            @Override
            public Long call(Long t1, Long t2) {
                System.out.println("t1 = " + t1 + ",t2 = " + t2);
                return t1 + t2;
            }
        }).subscribe(new Observer<Long>() {...});
Thread.sleep(100000);
```

4.join()类似combineLatest()，但是join操作符可以控制每个Observable产生结果的生命周期，在每个结果的生命周期内，可以与另一个Observable产生的结果按照一定的规则进行合并

```
Observable<Long> observable1 = Observable.interval(1000,
                TimeUnit.MILLISECONDS);
        Observable<Long> observable2 = Observable.interval(1000,
                TimeUnit.MILLISECONDS);
        observable1.join(observable2, new Func1<Long, Observable<Long>>() {
            @Override
            public Observable<Long> call(Long t) {
                System.out.println("left=" + t);
                return Observable.just(t).delay(1000, TimeUnit.MILLISECONDS);
            }
        }, new Func1<Long, Observable<Long>>() {
            @Override
            public Observable<Long> call(Long t) {
                System.out.println("right=" + t);
                return Observable.just(t).delay(1000, TimeUnit.MILLISECONDS);
            }
        }, new Func2<Long, Long, Long>() {
            @Override
            public Long call(Long t1, Long t2) {
                return t1 + t2;
            }
        }).subscribe(new Observer<Long>() {
            @Override
            public void onCompleted() {
                System.out.println("Observable  completed");
            }
 
            @Override
            public void onError(Throwable e) {
                System.out.println("Oh,no!  Something   wrong   happened！");
            }
 
            @Override
            public void onNext(Long t) {
                System.out.println("[result=]" + t);
            }
        });
 
        Thread.sleep(100000);
```

5.switchOnNext()把一组Observable转换成一个Observable，对于这组Observable中的每一个Observable所产生的结果，如果在同一个时间内存在两个或多个Observable提交的结果，只取最后一个Observable提交的结果给订阅者

```
Observable<Observable<Long>> observable = Observable.interval(2, TimeUnit.SECONDS)
        .map(new Func1<Long, Observable<Long>>() {
            @Override
            public Observable<Long> call(Long aLong) {
                return Observable.interval(1, TimeUnit.MILLISECONDS).take(5);
            }
        }).take(2);
 
Observable.switchOnNext(observable).subscribe(new Observer<Long>() {...});
Thread.sleep(1000000);
```

6.startWith()在Observable开始发射他们的数据之前，startWith()通过传递一个参数来先发射一个数据序列

```
Observable.just(1000, 2000).startWith(1, 2).subscribe(new Observer<Integer>() {...});

```

**总结**  
本文主要对rxjava进行了简单的介绍，从异步编程这个角度对rxjava进行了分析；并且针对Observable的过滤，转换，组合的API进行了简单的介绍，当然我们更关心的是rxjava有哪些应用场景。

**个人博客：[codingo.xyz](http://codingo.xyz/)**