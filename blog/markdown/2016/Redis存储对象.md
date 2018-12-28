      redis已经应用相当广泛了，但redis本身并没有直接存储对象的方法，我们可以通过转换对象的方式来存储对象。

大致总结了以下几种方案来存储对象：

**方案一：序列化对象为二进制**

使用redis的接口：

```java
jedis.get(byte[] key)
jedis.set(byte[] key, byte[] value)
```

至于序列化方式，我们有很多种选择，比如：Java serialize,Protobuf,或者自己手动序列化都行

```java
public byte[] serialize(Object obj);
public Object unSerialize(byte[] bytes);
```

**方案二：序列化对象为字符串**

使用redis的接口：

```java
jedis.get(String key);
jedis.set(String key, String value);
```

序列化为字符串，我们也有很多选择：Json(Jackson,FastJson),Xml等方式

**方案三：转换对象为Map**

使用redis的接口：

```java
jedis.hgetAll(String key);
jedis.hmset(String key, Map<String,String> values);
```

将对象转成一个map：

```java
/**
	 * 将指定的对象数据封装成map
	 * 
	 * @param bean
	 *            对象数据
	 * @return
	 */
	@SuppressWarnings("all")
	public static Map<String, String> warp(AbstractRedisBean bean) {
		Map<String, String> propertyMap = new HashMap<String, String>();
		try {
			PropertyDescriptor[] ps = Introspector.getBeanInfo(bean.getClass())
					.getPropertyDescriptors();
			for (PropertyDescriptor propertyDescriptor : ps) {
				String propertyName = propertyDescriptor.getName();
				if (propertyName != null && !propertyName.equals(CLASS)) {
					Method getter = propertyDescriptor.getReadMethod();
					if (getter != null) {
						RedisBeanField mannota = getter
								.getAnnotation(RedisBeanField.class);
						if (mannota != null && mannota.serialize() == false) {
							continue;
						}
						propertyMap.put(propertyName,
								String.valueOf(getter.invoke(bean, null)));
					}
				}
			}
		} catch (Exception e) {
			logger.error(e);
		}
		return propertyMap;
	}
```

将map转成java对象：

```java
/**
	 * 将map转成指定的对象
	 * 
	 * @param beanMap
	 *            map数据
	 * @param clazz
	 *            指定的类对象
	 * @return
	 */
	public static <T extends AbstractRedisBean> T reverse(
			Map<String, String> beanMap, Class<T> clazz) {
		T bean = getRedisBean(clazz);
		try {
			PropertyDescriptor[] ps = Introspector.getBeanInfo(clazz)
					.getPropertyDescriptors();
			for (PropertyDescriptor propertyDescriptor : ps) {
				String propertyName = propertyDescriptor.getName();
				if (propertyName != null && !propertyName.equals(CLASS)) {
					Method setter = propertyDescriptor.getWriteMethod();
					String value = beanMap.get(propertyName);
					String type = propertyDescriptor.getPropertyType()
							.getName();
					if (setter != null && value != null
							&& !value.equalsIgnoreCase("null")) {
						Object obj = value(value, type);
						if (obj != null) {
							setter.invoke(bean, obj);
						}
					}
				}
			}
		} catch (Exception e) {
			logger.error(e);
			e.printStackTrace();
		}
		bean.clearChangeMap();
		return bean;
	}

	/**
	 * 将String类型数值转换成指定的类型
	 * 
	 * @param value
	 *            数值
	 * @param type
	 *            指定的类型
	 * @return
	 */
	private static Object value(String value, String type) {
		if (type.equals("boolean")) {
			return Boolean.valueOf(value);
		} else if (type.equals("byte")) {
			return Byte.valueOf(value);
		} else if (type.equals("short")) {
			return Short.valueOf(value);
		} else if (type.equals("float")) {
			return Float.valueOf(value);
		} else if (type.equals("int")) {
			return Integer.valueOf(value);
		} else if (type.equals("java.lang.String")) {
			return value;
		} else if (type.equals("long")) {
			return Long.valueOf(value);
		}
		return null;
	}
```

**这种方式有一个优势：在更新对象的时候不需要更新整个对象，只需要更新需要的字段。**

**总结：大致总结了三种方案，每种方式都有自己的优势和劣势，根据不同的需求进行选择。**

**个人博客：[http://codingo.xyz](http://codingo.xyz/)**