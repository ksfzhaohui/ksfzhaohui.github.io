## 前言

上文[ASM入门篇](https://juejin.cn/post/6972350507100667935)中除了对`ASM`的使用做了介绍，同时也提到`ASM`被使用的一些场景，其中有一项就是`ASM`被用来代替`Java`反射；`FastJson`作为序列化工具，就使用了`ASM`来代替反射的使用，提高整体的性能。



## 序列化

序列化就是将对象转换成`Json`格式的字符串，然后用来持久化或者网络传输；`FastJson`提供了接口类`ObjectSerializer`：

```java
public interface ObjectSerializer {
    
    void write(JSONSerializer serializer, //
               Object object, //
               Object fieldName, //
               Type fieldType, //
               int features) throws IOException;
}
```

- serializer：序列化上下文；
- object：需要转换为Json的对象；
- fieldName：父对象字段名称；
- fieldType：父对象字段类型；
- features：父对象字段序列化features；

`ObjectSerializer`的实现类有多种，最常见就是序列化`JavaBean`的`JavaBeanSerializer`，当然还有一些其他类型的序列化工具类比如：基本类型、日期类型、枚举类型、一些特殊类型等等；引入的ASM正是对`JavaBeanSerializer`的优化，当然要使用ASM优化也是有条件的，满足条件的情况下会通过ASM生成`JavaBeanSerializer`的子类，代替里面的反射操作；

### 启用条件

FastJson提供了ASM的开关，默认开启状态，同时在满足一定的条件下才能使用`ASMSerializerFactory`创建`ASMSerializer`；

#### 开关

`SerializeConfig`中提供了asm开关标识：

```java
private boolean asm = !ASMUtils.IS_ANDROID
```

默认安卓环境下为`false`，否则为`true`，所有服务端开发一般都是开启asm的；

#### JSONType注解

- 如果配置了`serializer`，则使用配置的序列化类：

```java
@JSONType(serializer = JavaBeanSerializer.class)
```

- 如果配置了`asm`为`false`，则不启用`asm`：

```
@JSONType(asm = false)
```

- 如果配置了指定几种`SerializerFeature`，则不开启`asm`：

```java
@JSONType(serialzeFeatures = {SerializerFeature.WriteNonStringValueAsString,SerializerFeature.WriteEnumUsingToString,SerializerFeature.NotWriteDefaultValue,SerializerFeature.BrowserCompatible})
```

- 如果配置了`SerializeFilter`，则不开启`asm`：

```java
@JSONType(serialzeFilters = {})
```

#### BeanType类信息

如果类修饰符不为`public`，则不开启`asm`，直接使用`JavaBeanSerializer`；

如果类名称字符中包含了`<001 || >177 || ==.`的字符，则不启用`asm`；

如果类是接口类则不启用`asm`；

对类属性的检查，包括类型，返回值，注解等，不符合的情况下同样不启用`asm`；

### 创建ASMSerializer

通过`ASM`为每个`JavaBean`生成一个独立的`JavaBeanSerializer`子类，具体步骤如下：

#### 生成类名

创建类之前需要生成一个唯一的名称：

```java
String className = "ASMSerializer_" + seed.incrementAndGet() + "_" + clazz.getSimpleName()
```

这里的`seed`是一个`AtomicLong`变量；以`Person`为例，则生成的`className`为`ASMSerializer_1_Person`；

#### 生成子类

通过ASM的`ClassWriter`来生成`JavaBeanSerializer`的子类，重写`write`方法，`JavaBeanSerializer`中的`write`方法会使用反射从`JavaBean`中获取相关信息，而通过ASM生成的`ASMSerializer_1_Person`，是针对`Person`独有的序列化工具类，可以看部分代码：

```java
public class ASMSerializer_1_Person
extends JavaBeanSerializer
implements ObjectSerializer {
    public ASMSerializer_1_Person(SerializeBeanInfo serializeBeanInfo) {
        super(serializeBeanInfo);
    }

    public void write(JSONSerializer jSONSerializer, Object object, Object object2, Type type, int n) throws IOException {
        if (object == null) {
            jSONSerializer.writeNull();
            return;
        }
        SerializeWriter serializeWriter = jSONSerializer.out;
        if (!this.writeDirect(jSONSerializer)) {
            this.writeNormal(jSONSerializer, object, object2, type, n);
            return;
        }
        if (serializeWriter.isEnabled(32768)) {
            this.writeDirectNonContext(jSONSerializer, object, object2, type, n);
            return;
        }
        Person person = (Person)object;
        if (this.writeReference(jSONSerializer, object, n)) {
            return;
        }
        if (serializeWriter.isEnabled(0x200000)) {
            this.writeAsArray(jSONSerializer, object, object2, type, n);
            return;
        }
        SerialContext serialContext = jSONSerializer.getContext();
        jSONSerializer.setContext(serialContext, object, object2, 0);
        int n2 = 123;
        String string = "email";
        String string2 = person.getEmail();
        if (string2 == null) {
            if (serializeWriter.isEnabled(132)) {
                serializeWriter.write(n2);
                serializeWriter.writeFieldNameDirect(string);
                serializeWriter.writeNull(0, 128);
                n2 = 44;
            }
        } else {
            serializeWriter.writeFieldValueStringWithDoubleQuoteCheck((char)n2, string, string2);
            n2 = 44;
        }
        string = "id";
        int n3 = person.getId();
        serializeWriter.writeFieldValue((char)n2, string, n3);
        n2 = 44;
        string = "name";
        string2 = person.getName();
        if (string2 == null) {
            if (serializeWriter.isEnabled(132)) {
                serializeWriter.write(n2);
                serializeWriter.writeFieldNameDirect(string);
                serializeWriter.writeNull(0, 128);
                n2 = 44;
            }
        } else {
            serializeWriter.writeFieldValueStringWithDoubleQuoteCheck((char)n2, string, string2);
            n2 = 44;
        }
        if (n2 == 123) {
            serializeWriter.write(123);
        }
        serializeWriter.write(125);
        jSONSerializer.setContext(serialContext);
    }
    ...省略...
}
```

因为是仅是`Person`的序列化工具，所有可以发现里面直接强转`Object`为`Person`，通过直接调用的方式获取`Person`的相关信息，替换了反射的使用，我们知道直接调用的性能比使用反射强很多；

#### 查看源码

通过ASM生成的`JavaBeanSerializer`子类，转换成字节数组通过类加载直接加载到内存中，如果想查看自动生成的类源码可以使用如下两种方式来获取：

- 添加`Debug`代码
  在`ASMSerializerFactory`中找到`createJavaBeanSerializer`方法，ASM生成的代码最终会生成字节数组，部分代码如下所示：

  ```java
  byte[] code = cw.toByteArray();
  Class<?> serializerClass = classLoader.defineClassPublic(classNameFull, code, 0, code.length);
  ```

  在IDEA环境下可以在第二行处加断点，然后`右击`断点，选择`More`，勾选`Evaluate and log`，输入如下代码：

  ```java
  FileOutputStream fileOutputStream = new FileOutputStream(new File("F:/ASMSerializer_1_Person.class"));
  fileOutputStream.write(code);
  fileOutputStream.close();
  ```

- 使用`arthas`
  因为我们已经知道自动生成的类名，可以使用`arthas`监控当前进程，然后使用jad命令获取类源码：

  ```java
  [arthas@17916]$ jad com.alibaba.fastjson.serializer.ASMSerializer_1_Person
  
  ClassLoader:
  +-com.alibaba.fastjson.util.ASMClassLoader@32eebfca
    +-jdk.internal.loader.ClassLoaders$AppClassLoader@2437c6dc
      +-jdk.internal.loader.ClassLoaders$PlatformClassLoader@3688c050
  
  Location:
  /D:/myRepository/com/alibaba/fastjson/1.2.70/fastjson-1.2.70.jar
  
  /*
   * Decompiled with CFR.
   *
   * Could not load the following classes:
   *  com.fastjson.Person
   */
  package com.alibaba.fastjson.serializer;
  
  import com.alibaba.fastjson.serializer.JSONSerializer;
  import com.alibaba.fastjson.serializer.JavaBeanSerializer;
  import com.alibaba.fastjson.serializer.ObjectSerializer;
  import com.alibaba.fastjson.serializer.SerialContext;
  import com.alibaba.fastjson.serializer.SerializeBeanInfo;
  import com.alibaba.fastjson.serializer.SerializeWriter;
  import com.fastjson.Person;
  import java.io.IOException;
  import java.lang.reflect.Type;
  
  public class ASMSerializer_1_Person
  extends JavaBeanSerializer
  implements ObjectSerializer {
      public ASMSerializer_1_Person(SerializeBeanInfo serializeBeanInfo) {
  ```

  注意这里需要提供类的全名称：`包名+类名`

#### 加载类

通过ASM生成的类信息，不能直接使用，还需要通过类加载加载，这里通过`ASMClassLoader`来加载，加载完之后获取构造器`Constructor`，然后使用`newInstance`创建一个`JavaBeanSerializer`的子类：

```java
byte[] code = cw.toByteArray();
Class<?> serializerClass = classLoader.defineClassPublic(classNameFull, code, 0, code.length);
Constructor<?> constructor = serializerClass.getConstructor(SerializeBeanInfo.class);
Object instance = constructor.newInstance(beanInfo);
```



## 反序列化

将`Json`串转化成对象的过程，`FastJson`提供了`ObjectDeserializer`接口类：

```java
public interface ObjectDeserializer {
    <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName);
    int getFastMatchToken();
}
```

- parser：反序列化上下文DefaultJSONParser；
- type：要反序列化的对象的类型；
- fieldName：父对象字段名称；

同序列化过程，大致分为以下几个过程：

### 生成类名

生成一个业务类的反序列化工具类名：

```java
String className = "FastjsonASMDeserializer_" + seed.incrementAndGet() + "_" + clazz.getSimpleName();
```

同样以`Person`为例，那么生成的`className`为`FastjsonASMDeserializer_1_Person`；

### 生成子类

同样使用`ASM`的`ClassWriter`生成`JavaBeanDeserializer`的子类，重写其中的`deserialze`方法，部分代码如下：

```java
public class FastjsonASMDeserializer_1_Person
extends JavaBeanDeserializer {
    public char[] name_asm_prefix__ = "\"name\":".toCharArray();
    public char[] id_asm_prefix__ = "\"id\":".toCharArray();
    public char[] email_asm_prefix__ = "\"email\":".toCharArray();
    public ObjectDeserializer name_asm_deser__;
    public ObjectDeserializer email_asm_deser__;

    public FastjsonASMDeserializer_1_Person(ParserConfig parserConfig, JavaBeanInfo javaBeanInfo) {
        super(parserConfig, javaBeanInfo);
    }

    public Object createInstance(DefaultJSONParser defaultJSONParser, Type type) {
        return new Person();
    }

    public Object deserialze(DefaultJSONParser defaultJSONParser, Type type, Object object, int n) {
        block18: {
            String string;
            int n2;
            String string2;
            int n3;
            Person person;
            block20: {
                ParseContext parseContext;
                ParseContext parseContext2;
                block19: {
                    block22: {
                        JSONLexerBase jSONLexerBase;
                        block24: {
                            int n4;
                            int n5;
                            block23: {
                                block21: {
                                    String string3;
                                    String string4;
                                    jSONLexerBase = (JSONLexerBase)defaultJSONParser.lexer;
                                    if (jSONLexerBase.token() == 14 && jSONLexerBase.isEnabled(n, 8192)) {
                                        return this.deserialzeArrayMapping(defaultJSONParser, type, object, null);
                                    }
                                    if (!jSONLexerBase.isEnabled(512) || jSONLexerBase.scanType("com.fastjson.Person") == -1) break block18;
                                    ParseContext parseContext3 = defaultJSONParser.getContext();
                                    n5 = 0;
                                    person = new Person();
                                    parseContext2 = defaultJSONParser.getContext();
                                    parseContext = defaultJSONParser.setContext(parseContext2, person, object);
                                    if (jSONLexerBase.matchStat == 4) break block19;
                                    n4 = 0;
                                    n3 = 0;
                                    boolean bl = jSONLexerBase.isEnabled(4096);
                                    if (bl) {
                                        n3 |= 1;
                                        string4 = jSONLexerBase.stringDefaultValue();
                                    } else {
                                        string4 = null;
                                    }
                                    string2 = string4;
                                    n2 = 0;
                                    if (bl) {
                                        n3 |= 4;
                                        string3 = jSONLexerBase.stringDefaultValue();
                                    } else {
                                        string3 = null;
                                    }
                                    string = string3;
                                    string2 = jSONLexerBase.scanFieldString(this.email_asm_prefix__);
                                    if (jSONLexerBase.matchStat > 0) {
                                        n3 |= 1;
                                    }
                                    if ((n4 = jSONLexerBase.matchStat) == -1) break block20;
                                    if (jSONLexerBase.matchStat <= 0) break block21;
                                    ++n5;
                                    if (jSONLexerBase.matchStat == 4) break block22;
                                }
                                n2 = jSONLexerBase.scanFieldInt(this.id_asm_prefix__);
                                if (jSONLexerBase.matchStat > 0) {
                                    n3 |= 2;
                                }
                                if ((n4 = jSONLexerBase.matchStat) == -1) break block20;
                                if (jSONLexerBase.matchStat <= 0) break block23;
                                ++n5;
                                if (jSONLexerBase.matchStat == 4) break block22;
                            }
                            string = jSONLexerBase.scanFieldString(this.name_asm_prefix__);
                            if (jSONLexerBase.matchStat > 0) {
                                n3 |= 4;
                            }
                            if ((n4 = jSONLexerBase.matchStat) == -1) break block20;
                            if (jSONLexerBase.matchStat <= 0) break block24;
                            ++n5;
                            if (jSONLexerBase.matchStat == 4) break block22;
                        }
                        if (jSONLexerBase.matchStat != 4) break block20;
                    }
                    if ((n3 & 1) != 0) {
                        person.setEmail(string2);
                    }
                    if ((n3 & 2) != 0) {
                        person.setId(n2);
                    }
                    if ((n3 & 4) != 0) {
                        person.setName(string);
                    }
                }
                defaultJSONParser.setContext(parseContext2);
                if (parseContext != null) {
                    parseContext.object = person;
                }
                return person;
            }
            if ((n3 & 1) != 0) {
                person.setEmail(string2);
            }
            if ((n3 & 2) != 0) {
                person.setId(n2);
            }
            if ((n3 & 4) != 0) {
                person.setName(string);
            }
            return (Person)this.parseRest(defaultJSONParser, type, object, person, n, new int[]{n3});
        }
        return super.deserialze(defaultJSONParser, type, object, n);
    }
	...省略...
}
```

可以发现同样使用具体的业务类，代替了在`JavaBeanDeserializer`使用反射操作；同样最后需要对该类信息进行加载，然后通过构造器实例化具体业务对象。

## 总结

本文使用`ASM`代替反射只是众多使用场景的一种，在`FastJson`中只用了`ASM`的生成类功能；`ASM`更强的的功能是它的转换功能，对现有类进行改造，生成新的类从而对现有类进行功能增强，当然加载的过程也不是简单的使用类加载器就行了，这时候配合`Java Agent`来实现热更新；其实代替反射除了使用`ASM`，在`Protobuf`中也提供了一种方法，在研发阶段就把所有的业务序列化反序列化操作都准备好，是一种静态的处理方式，而`ASM`这种动态生成的方式更加人性化。