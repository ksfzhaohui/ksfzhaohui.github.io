## ASM简介

ASM是一个通用的Java字节码操作和分析框架，它可以用来修改现有的类或直接以二进制形式动态生成类。ASM提供了一些常见的字节码转换和分析算法，从中可以构建定制的复杂转换和代码分析工具。ASM提供了与其他Java字节码框架类似的功能，但侧重于性能。因为它的设计和实现都尽可能小和快，所以它非常适合在动态系统中使用（当然也可以以静态方式使用，例如在编译器中）。

ASM被用在很多项目中，包括如下：

- OpenJDK，生成lambda调用站点，以及Nashorn编译器；
- Groovy编译器和Kotlin编译器；
- Cobertura和Jacoco，以工具化类来度量代码覆盖率；
- CGLIB，用于动态生成代理类；
- Gradle，在运行时生成一些类；

更多参考官网：https://asm.ow2.io/

## IDE插件

ASM是直接对字节码进行操作，如果不熟悉字节码操作集合的话，写起来会很费劲，所以ASM为主流的IDE专门提供了开发插件BytecodeOutline：

- IDEA：[ASM Bytecode Outline](https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline/versions)
- Eclipse：[BytecodeOutline](http://download.forge.ow2.org/asm/)

以IDEA为例，只需要对应的类中右击->Show Bytecode outline即可，大致如下图所示：


![image-20210608154029529.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d456315c65d74d22acf5ebb5af68f56d~tplv-k3u1fbpfcp-watermark.image)

面板中包含三个页签：

- `Bytecode`：类对应的字节码文件；
- `ASMified`：利用`ASM`生成字节码对应的代码；
- `Groovified`：类对应的字节码指令；

## ASM API

`ASM`库提供了两个用于生成和转换已编译类的`API`，一个是核心`API`，以基于事件的形式来表示类；另一个是树`API`，以基于对象的形式来表示类；可以对比`XML`文件解析的方式：`SAX`模式和`DOM`模式；核心API对应`SAX`模式，树`API`对应`DOM`模式；每种模式都有自己的优缺点：

- 基于事件的API要快于基于对象的API，所需要的内存也较少，但在使用基于事件的API时，类转换的实现可能要更难一些；
- 基于对象的API会把整个类加载到内存中；

`ASM`库组织在几个包中，这些包分布在几个单独的JAR文件中：

- `org.objectweb.asm`和`org.objectweb.asm.signature`包：定义基于事件的API并提供类解析器和编写器组件，它们包含在asm.jar中；
- `org.objectweb.asm.util`包：提供基于核心API的各种工具，这些工具可在ASM应用程序的开发和调试过程中使用，包含在`asm-util.jar`中；
- `org.objectweb.asm.commons`包：提供了几个有用的预定义类转换器，主要基于核心API，包含在`asm-commons.jar`中；
- `org.objectweb.asm.tree`包：定义基于对象的API，并提供用于在基于事件的表示和基于对象的表示之间进行转换的工具，包含在`asm-tree.jar` 中；
- `org.objectweb.asm.tree.analysis`包：包提供了一个基于树API的类分析框架和几个预定义的类分析器，包含在`asm-analysis.jar`中；

## 核心API

在学习核心`API`之前，建议了解一下访问者模式，因为`ASM`对字节码的操作和分析都是基于访问者模式来实现的；

### 访问者模式

访问者模式建议将新行为放入一个名为*访问者*的独立类中， 而不是试图将其整合到已有类中。现在， 需要执行操作的原始对象将作为参数被传递给访问者中的方法， 让方法能访问对象所包含的一切必要数据；常见的应用场景：

- 如果你需要对一个复杂对象结构 （例如对象树） 中的所有元素执行某些操作， 可使用访问者模式；
- 可使用访问者模式来清理辅助行为的业务逻辑；
- 当某个行为仅在类层次结构中的一些类中有意义， 而在其他类中没有意义时， 可使用该模式；

字节码其实就是一个复杂的对象结构，还有像`Sharding-Jdbc`中对`sql`的解析也用到访问者模式，可以发现都是一些数据结构比较稳定的数据，固定的语法；

更多参考：[访问者模式](https://refactoringguru.cn/design-patterns/visitor)

### 类

访问者模式有两个核心类分别是：独立的访问者、接收访问者事件产生器；对应的`ASM`里面就是两个核心类：`ClassVisitor`和`ClassReader`，下面分别进行介绍；

#### ClassVisitor

用于生成和转换编译类的`ASM API`基于`ClassVisitor`抽象类，这个类中的每个方法都对应于同名的类文件结构：

```java
public abstract class ClassVisitor {
    public ClassVisitor(int api);
    public ClassVisitor(int api, ClassVisitor cv);
    public void visit(int version, int access, String name,String signature, String superName, String[] interfaces);
    public void visitSource(String source, String debug);
    public void visitOuterClass(String owner, String name, String desc);
    AnnotationVisitor visitAnnotation(String desc, boolean visible);
    public void visitAttribute(Attribute attr);
    public void visitInnerClass(String name, String outerName,String innerName, int access);
    public FieldVisitor visitField(int access, String name, String desc,String signature, Object value);
    public MethodVisitor visitMethod(int access, String name, String desc,String signature, String[] exceptions);
    void visitEnd();
}
```

内容可以具有任意长度和复杂性的部件将通过返回辅助访问者类，主要包括：`AnnotationVisitor`、`FieldVisitor`、`MethodVisitor`；更多可以参考`Java 虚拟机规范`；

以上所有方法都会被事件产生器`ClassReader`调用，所有方法中的参数都是`ClassReader`提供的，当然调用每个方法是有顺序的：

```
visit visitSource? visitOuterClass? ( visitAnnotation | visitAttribute )* ( visitInnerClass | visitField |visitMethod )* visitEnd
```

首先调用`visit`，然后是对`visitSource` 的最多一个调用，接下来是对`visitOuterClass` 的最多一个调用 ， 然后是可按任意顺序对 `visitAnnotation`和`visitAttribute`的任意多个访问 ， 接下来是可按任意顺序对 `visitInnerClass` 、`visitField` 和 `visitMethod` 的任意多个调用，最后以一个`visitEnd`调用结束。

#### ClassReader

此类主要功能就是读取字节码文件，然后把读取的数据通知`ClassVisitor`，字节码文件可以多种方式传入：

- `public ClassReader(final InputStream inputStream)`：字节流的方式；
- `public ClassReader(final String className)`：文件全路径；
- `public ClassReader(final byte[] classFile)`：二进制文件；

常见使用方式如下所示：

```java
ClassReader classReader = new ClassReader("com/zh/asm/TestService");
ClassWriter classVisitor = new ClassWriter(ClassWriter.COMPUTE_MAXS);
classReader.accept(classVisitor, 0);
```

`ClassReader`的`accept`方法处理接收一个访问者，还包括另外一个`parsingOptions`参数，选项包括：

- `SKIP_CODE`：跳过已编译代码的访问（如果您只需要类结构，这可能很有用）；
- `SKIP_DEBUG`：不访问调试信息，也不为其创建人工标签；
- `SKIP_FRAMES`：跳过堆栈映射帧；
- `EXPAND_FRAMES`：解压缩这些帧；

#### ClassWriter

以上实例中使用了`ClassWriter`，其继承于`ClassVisitor`，主要用来生成类，可以单独使用，如下所示：

```java
ClassWriter cw = new ClassWriter(0);
cw.visit(V1_5, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE,"pkg/Comparable", null, "java/lang/Object",new String[]{"pkg/Mesurable"});
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS","I", null, new Integer(-1)).visitEnd();
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL","I", null, new Integer(0)).visitEnd();
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER","I", null, new Integer(1)).visitEnd();
cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "compareTo","(Ljava/lang/Object;)I", null, null).visitEnd();
cw.visitEnd();
byte[] b = cw.toByteArray();

//输出
FileOutputStream fileOutputStream = new FileOutputStream(new File("F:/asm/Comparable.class"));
fileOutputStream.write(b);
fileOutputStream.close();
```

以上通过`ClassWriter`生成一个字节码文件，然后转换成字节数组，最后通过`FileOutputStream`输出到文件中，反编译结果如下：

```java
package pkg;

public interface Comparable extends Mesurable {
    int LESS = -1;
    int EQUAL = 0;
    int GREATER = 1;

    int compareTo(Object var1);
}
```

在实例化`ClassWriter`需要提供一个参数`flags`，选项包括：

- `COMPUTE_MAXS`：将为你计算局部变量与操作数栈部分的大小；还是必须调用 `visitMaxs`，但可以使用任何参数：它们将被忽略并重新计算；使用这一选项时，仍然必须自行计算这些帧；
- `COMPUTE_FRAMES`：一切都是自动计算；不再需要调用 `visitFrame`，但仍然必须调用 `visitMaxs`（参数将被忽略并重新计算）；
- 0：不会自动计算任何东西；必须自行计算帧、局部变量与操作数栈的大小；

以上只是对`ClassWriter`的单独使用，但更有意义的其实是把以上三个核心类整合起来使用，下面重点看看转换操作；

#### 转换操作

在类读取器和类写入器之间引入一个 `ClassVisitor`，把三者整合起来，大致代码结构如下所示：

```java
ClassReader classReader = new ClassReader("com/zh/asm/TestService");
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
//处理
ClassVisitor classVisitor = new AddFieldAdapter(classWriter...);
classReader.accept(classVisitor, 0);
```

上述代码相对应的体系结构如下图所示：


![image-20210609172035340.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3084d0d0c2df40649ca3d1339d428e1d~tplv-k3u1fbpfcp-watermark.image)

这里提供了一个添加属性的适配器，可以重写`visitEnd`方法，然后写入新的属性，代码如下：

```java
public class AddFieldAdapter extends ClassVisitor {
    private int fAcc;
    private String fName;
    private String fDesc;
    //是否已经有相同名称的属性
    private boolean isFieldPresent;

    public AddFieldAdapter(ClassVisitor cv, int fAcc, String fName,
                           String fDesc) {
        super(ASM4, cv);
        this.fAcc = fAcc;
        this.fName = fName;
        this.fDesc = fDesc;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String desc,
                                   String signature, Object value) {
        //判断是否有相同名称的字段，不存在才会在visitEnd中添加
        if (name.equals(fName)) {
            isFieldPresent = true;
        }
        return cv.visitField(access, name, desc, signature, value);
    }

    @Override
    public void visitEnd() {
        if (!isFieldPresent) {
            FieldVisitor fv = cv.visitField(fAcc, fName, fDesc, null, null);
            if (fv != null) {
                fv.visitEnd();
            }
        }
        cv.visitEnd();
    }
}
```

根据`ClassVisitor`的每个方法被调用的顺序，如果类中有多个属性，那么`visitField`会被调用多次，每次都会检查要添加的字段是否已经有了，然后保存在`isFieldPresent`标识中，这样在访问最后的`visitEnd`中判断是否需要添加新属性；

```java
ClassVisitor classVisitor = new AddFieldAdapter(classWriter,ACC_PUBLIC + ACC_FINAL + ACC_STATIC,"id","I");
```

这里添加了一个`public static final int id`；可以把字节数组写入class类文件中，然后反编译查看：

```java
public class TestService {
    public static final int id;
    ......
}
```

#### 工具类

除了上面几个核心类之外，`ASM`也提供了一些工具类，方便用户使用：

- **Type**
  `Type`对象表示一种 `Java`类型，既可以由类型描述符构造，也可以由`Class`对象构建；`Type`类还包含表示基元类型的静态变量；
- **TraceClassVisitor**
  扩展了`ClassVisitor`类，并构建了所访问类的文本表示；使用`TraceClassVisitor`以便获得实际生成内容的可读跟踪；
- **CheckClassAdapter**
  `ClassWriter` 类并不会核实对其方法的调用顺序是否恰当，以及参数是否有效；因此有可能会生成一些被 Java 虚拟机验证器拒绝的无效类。为了尽可能提前检测出部分此类错误，可以使用`CheckClassAdapter`类 ；
- **ASMifier**
  这个类为`TraceClassVisitor`工具提供了一个可选的后端（默认情况下，它使用一个`Textifier`后端，产生上面显示的输出类型）。这个后端使`TraceClassVisitor`类的每个方法打印用于调用它的Java代码。

### 方法

在介绍上面的`ClassVisitor`在访问复杂性的部件将通过返回辅助访问者类，其中包括：`AnnotationVisitor`、`FieldVisitor`、`MethodVisitor`；在介绍`MethodVisitor`之前了解一下Java 虚拟机执行模型；

#### 执行模型

每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等信
息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程；

- 局部变量表：包含可由其索引以随机顺序访问的变量；
- 操作数栈：字节码指令用作操作数的值堆栈；

看一个具有3帧的执行栈：

![image-20210610102350747.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afa9b7653b1d4931b838673f8ac264f1~tplv-k3u1fbpfcp-watermark.image)


第一帧：包含3个局部变量，操作数栈最大值为4，包含2个值；

第二帧：包含2个局部变量，操作数栈最大值为3，包含2个值；

第三帧：包含4个局部变量，操作数栈最大值为2，包含2个值；

#### 字节代码指令

字节码指令由标识该指令的操作码和固定数量的参数组成：

- 操作码：是一个无符号字节值，由助记符号标识。例如，操作码值0由助记符NOP设计，并对应于不执行任何操作的指令。
- 参数：是静态值，确定了精确的指令行为。它们紧跟在操作码之后给出。

字节码指令分为两类：

- 一小部分指令用于将值从局部变量转移到操作数堆栈；
- 其他指令只作用于操作数堆栈：它们从堆栈中弹出一些值，根据这些值计算结果，然后将其推回堆栈；

局部变量指令：

- `ILOAD`：用于加载一个 boolean、byte、 char、short 或int 局部变量；
- `LLOAD, FLOAD, DLOAD` ：分别用于加载 long、float 或 double值；
- `ALOAD`：用于加载任意非基元值，即对象和数组引用；

操作数栈指令：

- `ISTORE`：从操作数栈中弹出一个boolean、byte、 char、short 或int 局部变量值，并将它存储在由其索引i指定的局部变量中；
- `LSTORE，FSTORE，DSTORE`：分别弹出 long、float 或 double值；
- `ASTORE`：用于弹出任意非基元值；
- `GETFIELD`、`PUTFIELD`：`GETFIELD owner name desc`弹出一个对象引用，并推送其`name`字段的值；
  `PUTFIELD owner name desc`弹出一个值和一个对象引用，并将该值存储在其`name`字段中；
  在这两种情况下，对象必须是`owner`类型，其字段必须是`desc`类型。`GETSTATIC`和`PUTSTATIC`是类似的指令，但是对于静态字段。
- `INVOKEVIRTUAL、INVOKESTATIC、INVOKESPECIAL、INVOKEINTERFACE、INVOKEDYNAMIC`：
  `INVOKEVIRTUAL owner name desc`调用类`owner`中定义的`name`方法，其方法描述符为`desc`。`INVOKESTATIC`用于静态方法，`INVOKESPECIAL`用于私有方法和构造函数，`INVOKEINTERFACE`用于接口中定义的方法。最后，对于java7类，`INVOKEDYNAMIC`用于新的动态方法调用机制。

#### MethodVisitor

用于生成和转换已编译方法的`ASM API`是基于`MethodVisitor`抽象类的；它由`ClassVisitor`的`visitMethod`方法返回；此类还根据这些指令的参数数量和类型为每个字节码指令类别定义了一个方法；必须按以下顺序调用这些方法：

```java
visitAnnotationDefault? ( visitAnnotation | visitParameterAnnotation | visitAttribute )*( visitCode( visitTryCatchBlock | visitLabel | visitFrame | visitXxx Insn |visitLocalVariable | visitLineNumber )*visitMaxs )?visitEnd
```

下面看一个对现有方法进行转换实例，给方法添加开始和结束日志；

1. 准备需要被转换的实例，在`query`方法处理前和处理后添加日志；

   ```java
   public class TestService {
   	public void query(int param) {
   		System.out.println("service handle...");
   	}
   }
   ```

2. 重写`ClassVisitor`中的`visitMethod`

   ```java
   public class MyClassVisitor extends ClassVisitor implements Opcodes {
       public MyClassVisitor(ClassVisitor cv) {
           super(ASM5, cv);
       }
   
       @Override
       public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
           MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature,
                   exceptions);
           if (!name.equals("<init>") && methodVisitor != null) {
               methodVisitor = new MyMethodVisitor(methodVisitor);
           }
           return methodVisitor;
       }
   }
   ```

   过滤掉`<init>`方法，其他方法都会被`MyMethodVisitor`包装，然后重写`MethodVisitor`的方法；

3. 重载MethodVisitor

   ```java
   public class MyMethodVisitor extends MethodVisitor implements Opcodes {
       public MyMethodVisitor(MethodVisitor mv) {
           super(Opcodes.ASM4, mv);
       }
   
       @Override
       public void visitCode() {
           super.visitCode();
           mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
           mv.visitLdcInsn("start");
           mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
       }
   
       @Override
       public void visitInsn(int opcode) {
           if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN)
                   || opcode == Opcodes.ATHROW) {
               //方法在返回之前打印"end"
               mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
               mv.visitLdcInsn("end");
               mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
           }
           mv.visitInsn(opcode);
       }
   }
   ```

   `visitCode`方法访问之前调用，`visitInsn`需要判断操作符是不是方法返回，一般方法在返回之前会执行`mv.visitInsn(RETURN)`操作，这时候可以通过`opcode`来判断；

4. 查看生成的新的字节码文件

   ```java
   public class TestService {
       public TestService() {
       }
   
       public void query(int var1) {
           System.out.println("start");
           System.out.println("service handle...");
           System.out.println("end");
       }
   }
   ```

   

#### 工具类

在方法下面也同样提供了一些工具类：

- `LocalVariablesSorter`：此方法适配器将一个方法中使用的局部变量按照它们在这个方法中的出现顺序重新进行编号，同时可以使用 `newLocal` 方法创建一个新的局部变量；
- `AdviceAdapter`：此方法适配器是一个抽象类，可用于在方法的开头以及任何`RETURN`或`ATHROW`指令之前插入代码；其主要优点是它也适用于构造函数，其中代码不能仅插入构造函数的开头，而是在调用超级构造函数之后插入。

## 使用场景

ASM被用在很多项目中，这里介绍两种常见的使用场景：AOP和代替反射；

### AOP

面向切面编程，在程序开发中主要用来解决一些系统层面上的问题，比如日志，事务，权限等待；其中关键技术就是代理，代理包括动态代理和静态代理，实现的方式也有多种：

- AspectJ：属于静态织入，原理是静态代理；
- JDK动态代理：`JDK`动态代理两个核心类：`Proxy`和`InvocationHandler`；
- Cglib动态代理：封装了`ASM`，可以再运行期动态生成新的`Class`；功能上比`JDK动态代理`更强大；

其中的`Cglib`动态代理方式就依赖`ASM`，上面的实例中我们也看到了`ASM`的字节码增强功能；

### 代替反射

`FastJson`以速度快著称，其中有一项就是使用`ASM`代替了`Java`反射；另外还有`ReflectASM`包专门用来代替`Java`反射；

> ReflectASM 是一个非常小的 Java 类库，通过代码生成来提供高性能的反射处理，自动为 get/set 字段提供访问类，访问类使用字节码操作而不是 Java 的反射技术，因此非常快。

看一段`ReflectASM`简单使用方式：

```java
TestBean testBean = new TestBean(1, "zhaohui", 18);
MethodAccess methodAccess = MethodAccess.get(TestBean.class);
String[] mns = methodAccess.getMethodNames();

for (int i = 0; i < mns.length; i++) {
    System.out.println(methodAccess.invoke(testBean, mns[i]));
}
```

这里正常打印`TestBean`中的属性值，为什么速度快，因为内部会通过`ASM`生成一个临时的`TestBeanMethodAccess`，内部重写了invoke方法，反编译之后如下所示：

```java
public Object invoke(Object var1, int var2, Object... var3) {
        TestBean var4 = (TestBean)var1;
        switch(var2) {
        case 0:
            return var4.getName();
        case 1:
            return var4.getId();
        case 2:
            return var4.getAge();
        default:
            throw new IllegalArgumentException("Method not found: " + var2);
        }
 }
```

可以发现invoke里面其实就是普通的调用，速度肯定比使用java反射快。

## 参考文档

[asm4-guide.pdf](https://asm.ow2.io/asm4-guide.pdf)

[ASM4手册中文版](http://www.java1234.com/a/javabook/javabase/2020/0408/16198.html)

## 感谢关注

> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。