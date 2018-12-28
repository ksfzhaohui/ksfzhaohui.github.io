        使用 Instrumentation，开发者可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在 JVM 上的程序，甚至能够替换和修改某些类的定义。有了这样的功能，开发者就可以实现更为灵活的运行时虚拟机监控和 Java 类操作了，这样的特性实际上提供了一种虚拟机级别支持的 AOP 实现方式，使得开发者无需对 JDK 做任何升级和改动，就可以实现某些 AOP 的功能了。

       网上的基于Instrumentation的说明有很多，不想做过多的介绍，下面基于项目中遇到的一些需求，做的一个简单的例子。

       做的项目是一个手机网络游戏，游戏的业务变更性很强，特别是项目开发中以及项目上线后的前期，有些东西变更很少，但对游戏来说很重要，比如掉率，或者一些其他的数值影响，这对程序来说可能就是该一个数值，这种情况下如果去重启服务器有点得不偿失，如果能进行热替换就好了。下面看一个定时检查指定文件夹下需要变更的文件，然后进行热替换。

1.首先我们需要提供一个有premain方法的类

```java
package com.hotpatch;

import java.lang.instrument.Instrumentation;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import org.apache.log4j.Logger;

/**
 * 热替换
 * 
 * @author ksfzhaohui
 * 
 */
public class HotPatch {

	private final static Logger logger = Logger.getLogger(HotPatch.class);

	public static void premain(String agentArgs, Instrumentation inst) {
		Executors.newScheduledThreadPool(1).scheduleAtFixedRate(
				new HotPatchThread(inst), 5, 5, TimeUnit.SECONDS);
		logger.info("hotPatch starting...");
	}
}

```

提供了一个premain的方法，并且启动了一个定时器，定时执行HotPatchThread线程。

2.HotPatchThread线程

```java
package com.hotpatch;

import java.io.File;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;
import java.util.List;
import org.apache.log4j.Logger;

/**
 * 热替换线程
 * 
 * @author ksfzhaohui
 * 
 */
public class HotPatchThread implements Runnable {

	private final static Logger logger = Logger.getLogger(HotPatchThread.class);
	private static final String ROOT_PATH = "hotfiles";
	private Instrumentation inst;

	public HotPatchThread(Instrumentation inst) {
		this.inst = inst;
	}

	public void run() {
		try {
			List<File> list = FileUtil.readfile(ROOT_PATH);
			if (list != null && list.size() > 0) {
				for (File file : list) {
					Class<?> clazz = Class.forName(getPackageName(file));
					byte[] array = FileUtil.getBytesFromFile(file.getPath());
					ClassDefinition def = new ClassDefinition(clazz, array);
					inst.redefineClasses(def);

					file.delete();
					logger.info("hotpatch " + file.getPath() + " success");
				}
			}
		} catch (Exception e) {
			logger.error("hotpatching error", e);
		}
	}

	/**
	 * 获取类的包名+类名
	 * 
	 * @param file
	 * @return
	 */
	private String getPackageName(File file) {
		String path = file.getPath();
		int index = path.indexOf(ROOT_PATH);
		path = path.substring(index + ROOT_PATH.length() + 1);
		path = path.split("\\.")[0];
		path = path.replaceAll("\\\\", ".");
		return path;
	}
}

```

HotPatchThread线程定时读取hotfiles路径下的需要更新的文件，然后进行热替换，  
注：包名需要以文件夹的形式存在

下面看一下代码结构：

![](http://static.oschina.net/uploads/space/2016/0521/214510_EKFJ_159239.png)    结构很简单，就3个类实现简单的热替换

源码地址:[https://github.com/ksfzhaohui/hotpatch.git](https://github.com/ksfzhaohui/hotpatch.git)

下面进行简单的测试：

```java
package agentTest;

public class AgentTest {

	public static void main(String[] args) throws InterruptedException {
		TClass c = new TClass();
		while (true) {
			System.out.println(c.getNumber());
			Thread.sleep(1000);
		}
	}
}
```

每隔一秒读取一下number

```java
package agentTest;

public class TClass {

	private int k = 10;

	public int getNumber() {
		return k + 4;
	}
}
```

将程序打成jar包例如叫test.jar，然后可以写一个bat文件：run.bat  
 

```bash
java -javaagent:hotpatch-0.0.1-SNAPSHOT.jar -cp test.jar agentTest.AgentTest 
```

下面修改一下TClass文件，把k+4改成k+5，编译成class文件，放在hotfiles文件夹下

![](http://static.oschina.net/uploads/space/2016/0521/215250_w0wl_159239.png)

hotfiles文件夹下有一个agentTest文件夹(TClass的包名)，agentTest下面是改变过的TClass.class文件

运行结果：

![](http://static.oschina.net/uploads/space/2016/0521/215813_T6lz_159239.png)

总结：总体来说可以实现一些简单的需求，也只限于对已有的方法进行修改，对于整个类的结构修改，仍然需要重启虚拟机