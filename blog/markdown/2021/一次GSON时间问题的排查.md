本文正在参加「Java主题月 - Java Debug笔记活动」，详情查看<[活动链接](https://juejin.cn/post/6960478432744931364/)>


## 问题描述

近期有一个老项目在测试环境中频繁出现了GSON反序列化时间问题，错误堆栈如下所示：

```java
Exception in thread "main" com.google.gson.JsonSyntaxException: 2021-05-14 14:59:37
	at com.google.gson.internal.bind.DateTypeAdapter.deserializeToDate(DateTypeAdapter.java:81)
	at com.google.gson.internal.bind.DateTypeAdapter.read(DateTypeAdapter.java:66)
	at com.google.gson.internal.bind.DateTypeAdapter.read(DateTypeAdapter.java:41)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$1.read(ReflectiveTypeAdapterFactory.java:93)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:172)
	at com.google.gson.Gson.fromJson(Gson.java:795)
	at com.google.gson.Gson.fromJson(Gson.java:761)
	at com.google.gson.Gson.fromJson(Gson.java:710)
	at com.google.gson.Gson.fromJson(Gson.java:682)
	at com.gson.GsonDate.main(GsonDate.java:17)
Caused by: java.text.ParseException: Unparseable date: "2021-05-14 14:59:37"
	at java.text.DateFormat.parse(DateFormat.java:366)
	at com.google.gson.internal.bind.DateTypeAdapter.deserializeToDate(DateTypeAdapter.java:79)
	... 9 more
```

错误描述也很详细，就是GSON在反序列化一段Json串的时候，因为某个时间字符串无法反序列化，导致最终整个Json反序列化失败；

### 背景说明

因为此系统数据量比较大，所有会将半年前的数据归档到Hbase中，归档的时候会将数据库中的数据序列化为json格式，然后保存到Hbase中；如果是近期半年的数据会直接查询数据库，如果是很早的数据才会查询Hbase，所以出现的概率比较低；



## 问题分析

### 本地重现

为了方便分析，直接把Json串拷贝到本地，然后再本地进行重现，再进行问题分析，Json串比较长，这里使用如下Json串代替：

```json
{"date":"2021-05-14 14:59:37"}
```

准备相关代码如下所示：

```java
public class GsonDate {
	public static void main(String[] args) {
		String json = "{\"date\":\"2021-05-14 14:59:37\"}";
		GsonDateBean date = new Gson().fromJson(json, GsonDateBean.class);
		System.out.println(date);
	}
}

@Data
class GsonDateBean {
	private Date date;
}
```

执行的结果是可以反序列成功，并没有出现上面的错误，为了找出原因，这里需要分析一下Gson时间转换的相关源码；

### 源码分析

Gson时间转换的源码还是比较简单的，`DateTypeAdapter`部分代码如下所示：

```java
  private final DateFormat enUsFormat
      = DateFormat.getDateTimeInstance(DateFormat.DEFAULT, DateFormat.DEFAULT, Locale.US);
  private final DateFormat localFormat
      = DateFormat.getDateTimeInstance(DateFormat.DEFAULT, DateFormat.DEFAULT);
  private final DateFormat iso8601Format = buildIso8601Format();

  private static DateFormat buildIso8601Format() {
    DateFormat iso8601Format = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'", Locale.US);
    iso8601Format.setTimeZone(TimeZone.getTimeZone("UTC"));
    return iso8601Format;
  }

  private synchronized Date deserializeToDate(String json) {
    try {
      return localFormat.parse(json);
    } catch (ParseException ignored) {
    }
    try {
      return enUsFormat.parse(json);
    } catch (ParseException ignored) {
    }
    try {
      return iso8601Format.parse(json);
    } catch (ParseException e) {
      throw new JsonSyntaxException(json, e);
    }
  }
```

Gson准备了三个`DateFormat`，分别是：localFormat，enUsFormat，iso8601Format；转换的时候也是按照这个顺序进行转换，哪个能转换成功就直接返回，以上出现问题说明三种`DateFormat`都没有转换成功；本地调试可以直接Debug进来，可以发现直接使用localFormat就转换成功了，并且可以分别查看每个的`pattern`；

- localFormat：yyyy-M-d H:mm:ss
- enUsFormat：MMM d, yyyy h:mm:ss a
- iso8601Format：yyyy-MM-dd'T'HH:mm:ss'Z'

以上的日期格式完全符合`yyyy-M-d H:mm:ss`格式，所以可以直接转换成功；可以发现localFormat其实是和本地系统的语言环境有关，所以会出现本地运行结果和服务器运行结果不一致；

### 再次重现

可以直接通过代码设置语言环境，把环境设置为`Locale.US`

```java
public class GsonDate {
	public static void main(String[] args) {
		System.out.println("默认："+Locale.getDefault());
		System.out.println("重置语言环境：Locale.US");
		Locale.setDefault(Locale.US);
		String json = "{\"date\":\"2021-05-14 14:59:37\"}";
		GsonDateBean date = new Gson().fromJson(json, GsonDateBean.class);
		System.out.println(date);
	}
}
```

运行以上代码，出现了和服务器一样的反序列时间问题：

```java
默认：zh_CN
重置语言环境：Locale.US
Exception in thread "main" com.google.gson.JsonSyntaxException: 2021-05-14 14:59:37
	at com.google.gson.internal.bind.DateTypeAdapter.deserializeToDate(DateTypeAdapter.java:81)
```

可以发现我们本地的环境一般都是`zh_CN`，对应`Locale.CHINA`；

### 问题解决

#### 系统配置

可以直接改变系统语言环境，liunx可以直接在`/etc/sysconfig/i18n`中配置：

```shell
英文版系统：
LANG="en_US.UTF-8"
中文版系统：
LANG="zh_CN.UTF-8"
```

可以查看当前配置的语言环境：

```shell
[root@Centos ~]# echo $LANG
en_US.UTF-8
```

#### 代码实现

可以给Gson设置默认的日志转换格式：

```java
Gson gson = new GsonBuilder().setDateFormat("yyyy-MM-dd HH:mm:ss").create();
GsonDateBean date = gson.fromJson(json, GsonDateBean.class);
```

## 扩展

同样的如果使用其他Json序列化工具，比如`fastjson`是否也有这样的问题那，可以简单做一个测试：

```
Locale.setDefault(Locale.US);
String json = "{\"date\":\"2021-05-14 14:59:37\"}";
String json2 = "{\"date\":\"2021年05月14日 14:59:37\"}";
JacksonDateBean date = JSON.parseObject(json, JacksonDateBean.class);
```
结果是不仅`yyyy-MM-dd HH:mm:ss`格式能被解析，包含中文的`年月日`都可以被解析；如果查看相关源码可以发现，`fastjson`并没有直接使用`DateFormat`去做日期格式转换，而是实现了[ISO 8601标准](https://zh.wikipedia.org/zh-hans/ISO_8601)，并且提供了中国常见日期格式的支持；具体可以直接查看源码`JSONScanner`中的`scanISO8601DateIfMatch`方法；
另外一点需要说明的是以上GSON使用的是`2.2.2`版本，最新版本`2.8.6`版本中同样提供了对`ISO 8601标准`的支持，具体可以查看`ISO8601Utils`类。

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。