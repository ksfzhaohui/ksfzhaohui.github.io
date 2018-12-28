**前言**  
最近测试经常反应游戏中出现随机的地方，比如：开宝箱，装备的掉落以及属性的随机等，表现的不尽如意；所以开始怀疑我们的随机算法，而我们使用的就是JDK自带的Random类，跟他们解释他们也不太明白，没办法只能以一种更加直观的方式展示给他们看更具有说服力，刚好也可以更加深入的了解一下Random类。

**简介**  
打开Random类的源代码，在类注释中可以看到如下说明：  
此类的实例用于生成伪随机数流。此类使用48位的种子，使用线性同余公式 (linear congruential form) 对其进行了修改（请参阅 Donald Knuth 的The Art of Computer Programming, Volume 3，第 3.2.1 节）。  
如果用相同的种子创建两个Random实例，则对每个实例进行相同的方法调用序列，它们将生成并返回相同的数字序列。为了保证此属性的实现，为类Random指定了特定的算法。为了Java代码的完全可移植性，Java实现必须让类Random使用此处所示的所有算法。但是允许Random类的子类使用其他算法，只要其符合所有方法的常规协定即可。  
Random类实现的算法使用一个protected实用工具方法，每次调用它最多可提供32个伪随机生成的位。

里面提到了**线性同余公式**，是个产生伪随机数的方法，下面会进行介绍，先看一下Random提供了哪些方法：

**Random方法介绍**  
int next(int bits)：生成下一个伪随机数。  
boolean nextBoolean()：返回下一个伪随机数，它是取自此随机数生成器序列的均匀分布的boolean值。  
void nextBytes(byte\[\] bytes)：生成随机字节并将其置于用户提供的 byte 数组中。  
double nextDouble()：返回下一个伪随机数，它是取自此随机数生成器序列的、在0.0和1.0之间均匀分布的 double值。  
float nextFloat()：返回下一个伪随机数，它是取自此随机数生成器序列的、在0.0和1.0之间均匀分布float值。  
double nextGaussian()：返回下一个伪随机数，它是取自此随机数生成器序列的、呈高斯（“正态”）分布的double值，其平均值是0.0标准差是1.0。  
int nextInt()：返回下一个伪随机数，它是此随机数生成器的序列中均匀分布的 int 值。  
int nextInt(int n)：返回一个伪随机数，它是取自此随机数生成器序列的、在（包括和指定值（不包括）之间均匀分布的int值。  
long nextLong()：返回下一个伪随机数，它是取自此随机数生成器序列的均匀分布的 long 值。  
void setSeed(long seed)：使用单个 long 种子设置此随机数生成器的种子。

通过上面的介绍，大致可以对Random做如下总结：  
1.Random是伪随机，既随机是有规则的，  
2.相同的种子创建两个Random实例，则对每个实例进行相同的方法调用序列，它们将生成并返回相同的数字序列，  
3.Random类中的各方法生成的随机数字都是均匀分布的。

下面通过实例进行简单的验证：

**实例验证**  
1.Random是伪随机，既随机是有规则的

```
public class RandomTest {
 
    public static void main(String[] args) {
        Random r = new Random(10);
        for (int i = 0; i < 10; i++) {
            System.out.println("rv = " + r.nextInt(100));
        }
    }
}
```

多次运行RandomTest ，发现每次输出的结果都是一样的，说明随机是有规则的

2.相同的种子创建两个Random实例，则对每个实例进行相同的方法调用序列，它们将生成并返回相同的数字序列

```
public class RandomTest {
 
    public static void main(String[] args) {
        Random r1 = new Random(10);
        for (int i = 0; i < 10; i++) {
            System.out.println("rv1 = " + r1.nextInt(100));
        }
        System.out.println("========分割线==========");
        Random r2 = new Random(10);
        for (int i = 0; i < 10; i++) {
            System.out.println("rv2 = " + r2.nextInt(100));
        }
    }
}
```

分别设置了相同的种子，r1和r2运行的结果是一样的

3.Random类中的各方法生成的随机数字都是均匀分布的

```
public class TestRandom extends ApplicationFrame {
 
    private static final long serialVersionUID = 1L;
 
    private static final int COUNT = 500000;
 
    public TestRandom(final String title) {
        super(title);
        float[][] data = initData();
        NumberAxis domainAxis = new NumberAxis("X");
        domainAxis.setAutoRangeIncludesZero(false);
        NumberAxis rangeAxis = new NumberAxis("Y");
        rangeAxis.setAutoRangeIncludesZero(false);
 
        FastScatterPlot plot = new FastScatterPlot(data, domainAxis, rangeAxis);
        JFreeChart chart = new JFreeChart("Test Random", plot);
        chart.getRenderingHints().put(RenderingHints.KEY_ANTIALIASING,
                RenderingHints.VALUE_ANTIALIAS_ON);
 
        ChartPanel panel = new ChartPanel(chart, true);
        panel.setPreferredSize(new Dimension(800, 500));
        panel.setMinimumDrawHeight(10);
        panel.setMaximumDrawHeight(2000);
        panel.setMinimumDrawWidth(20);
        panel.setMaximumDrawWidth(2000);
 
        setContentPane(panel);
    }
 
    private float[][] initData() {
        float[][] data = new float[2][COUNT];
        Random random = new Random();
        for (int i = 0; i < COUNT; i++) {
            data[0][i] = i + 100000;
            data[1][i] = 100000 + random.nextInt(500000);
        }
        return data;
    }
 
    public static void main(final String[] args) {
        TestRandom demo = new TestRandom("Test Random");
        demo.pack();
        RefineryUtilities.centerFrameOnScreen(demo);
        demo.setVisible(true);
    }
 
}
```

上面的类依赖于第三方库JFreeChart，用来生成图表的，生成的图片如下图所示：

![](https://static.oschina.net/uploads/space/2017/0110/220641_o5ru_159239.jpg)

从图片中可以看到，随机生成的点还是挺均匀的，可以丢给测试去看了。

**线性同余公式**  
线性同余公式LCG(linear congruential generator)是个产生伪随机数的方法，它的优点是：速度快，内存消耗少。  
LCG 算法数学上基于公式（来源wiki）：  
**N(j+1) = (A * N(j) + B)(mod M)**  
其中N(j)是序列的第j个数，N(j+1)是序列的第j+1个数，变量A，B，M是常数，A是乘数，B是增量，M是模，初始值为N(0)。  
LCG的周期最大为M，但大部分情况都会少于M，M一般都设置的很大，在选择常数时需要注意，以保证能找到最大的周期；要令LCG达到最大周期，应符合以下条件：  
1.B,M互质；  
2.M的所有质因数都能整除A-1；  
3.若M是4的倍数，A-1也是；  
4.A,B,N(0)都比M小；  
5.A,B是正整数。

**Random类中实现的线性同余算法**  
1.Random的两种构造方法：  
public Random()  
public Random(long seed)  
第一个构造方法没有指定种子，则系统默认使用和系统时间相关的种子，如下所示：

```
public Random() {
      this(seedUniquifier() ^ System.nanoTime());
}
 
private static long seedUniquifier() {
      for (;;) {
          long current = seedUniquifier.get();
          long next = current * 181783497276652981L;
          if (seedUniquifier.compareAndSet(current, next))
              return next;
     }
}
 
private static final AtomicLong seedUniquifier
        = new AtomicLong(8682522807148012L);
```

seedUniquifier这个方法有没有似曾相识，和AtomicLong里面的incrementAndGet一样：

```
public final long incrementAndGet() {
      for (;;) {
          long current = get();
          long next = current + 1;
          if (compareAndSet(current, next))
              return next;
      }
}
```

其实就是以原子的方式对seedUniquifier乘以181783497276652981L，然后和System.nanoTime()进行异或运算，得到最终的种子，保证了默认种子的随机性。至于为什么有8682522807148012L和181783497276652981L这两个数值，可以参考这个[What’s with 181783497276652981 and 8682522807148012 in Random (Java 7)?](http://stackoverflow.com/questions/18092160/whats-with-181783497276652981-and-8682522807148012-in-random-java-7)

第二个构造方法由用户自己指定一个种子，伪随机数都是基于一个种子数的，然后每需要一个随机数，都是对当前种子进行一些数学运算，得到一个数，基于这个数得到需要的随机数和新的种子。

2.有了种子数之后，其他数是怎么生成的，看最核心的方法next(int bits)，代码如下：

```
private static final long multiplier = 0x5DEECE66DL;
private static final long addend = 0xBL;
private static final long mask = (1L << 48) - 1;
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

最重要的一段代码就是：  
nextseed = (oldseed * multiplier + addend) & mask;  
简单来讲就是用旧的种子(oldseed)乘以一个数(multiplier)，加上一个数addend，然后取低48位作为结果(mask相与)。这段代码其实就是上面介绍的**线性同余公式**，用来帮我们产生伪随机数。

**总结**  
总结一句话就是：Random以线性同余公式为核心算法，基于一个种子数，产生伪随机数。