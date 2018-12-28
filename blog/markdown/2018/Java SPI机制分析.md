##SPI����
SPIȫ��Ϊ(Service Provider Interface) ����JDK���õ�һ�ַ����ṩ���ֻ��ƣ���Ҫ����ܵĿ�����Աʹ�ã�����java.sql.Driver�ӿڣ����ݿ⳧��ʵ�ִ˽ӿڼ��ɣ���ȻҪ����ϵͳ֪������ʵ����Ĵ��ڣ�����Ҫʹ�ù̶��Ĵ�Ź�����Ҫ��classpath�µ�META-INF/services/Ŀ¼�ﴴ��һ���Է���ӿ��������ļ�������ļ�������ݾ�������ӿڵľ����ʵ���ࣻ������JDBCΪʵ�������о���ķ�����

##JDBC����
###1.׼��������

```
<dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.2.2</version>
        </dependency>
        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>mssql-jdbc</artifactId>
            <version>7.0.0.jre8</version>
        </dependency>
```
�ֱ�׼����mysql��postgresql��sqlserver�����Դ�jar������ÿ��jar����META-INF/services/������һ��java.sql.Driver�ļ����ļ��������һ����������������mysql��

```
com.mysql.jdbc.Driver
com.mysql.fabric.jdbc.FabricMySQLDriver
```
�ṩ��ÿ��������ռ��һ�У�������ʱ��ᰴ�ж�ȡ������ʹ���ĸ������url��������

###2.��ʵ��

```
String url = "jdbc:mysql://localhost:3306/db3";
String username = "root";
String password = "root";
String sql = "update travelrecord set name=\'bbb\' where id=1";
Connection con = DriverManager.getConnection(url, username, password);
```
��·���´��ڶ����������������ʹ��DriverManager.getConnectionӦ��ʹ���ĸ�����������url��ʶ�𣬲�ͬ�����ݿ��в�ͬ��urlǰ׺��

###3.��������ط���
����META-INF/services/�µ���������ʲôʱ����صģ�DriverManager��һ����̬����飺

```
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
 
private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    // If the driver is packaged as a Service Provider, load it.
    // Get all the drivers through the classloader
    // exposed as a java.sql.Driver.class service.
    // ServiceLoader.load() replaces the sun.misc.Providers()
 
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
 
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
 
            /* Load these drivers, so that they can be instantiated.
             * It may be the case that the driver class may not be there
             * i.e. there may be a packaged driver with the service class
             * as implementation of java.sql.Driver but the actual class
             * may be missing. In that case a java.util.ServiceConfigurationError
             * will be thrown at runtime by the VM trying to locate
             * and load the service.
             *
             * Adding a try catch block to catch those runtime errors
             * if driver not available in classpath but it's
             * packaged as service and that service is there in classpath.
             */
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });
 
    println("DriverManager.initialize: jdbc.drivers = " + drivers);
 
    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```
�ڼ���DriverManager���ʱ���ִ��loadInitialDrivers������������ͨ�������ּ���������ķ�ʽ���ֱ��ǣ�ʹ��ϵͳ������ʽ��ServiceLoader���ط�ʽ��ϵͳ������ʽ��ʵ�����ڱ���jdbc.drivers�����ú������࣬Ȼ��ʹ��Class.forName���м��أ������ص㿴һ��ServiceLoader��ʽ���˴�������load�������ǲ�û������ȥ���������࣬���Ƿ�����һ��LazyIterator������Ĵ������ѭ��������������

```
private static final String PREFIX = "META-INF/services/";
 
private class LazyIterator
        implements Iterator<S>
    {
 
        Class<S> service;
        ClassLoader loader;
        Enumeration<URL> configs = null;
        Iterator<String> pending = null;
        String nextName = null;
 
        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }
 
        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }
 
        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
        ......
    }
```
����ָ����һ����̬����PREFIX = ��META-INF/services/����Ȼ���java.sql.Driverƴ�������fullName��Ȼ��ͨ���������ȥ��ȡ������·����java.sql.Driver�ļ�����ȡ֮������configs�У������ÿ��Ԫ�ض�Ӧһ���ļ���ÿ���ļ��п��ܻ���ڶ�������࣬����ʹ��pending�������ÿ���ļ��е�������Ϣ����ȡ������Ϣ֮����nextService��ʹ��Class.forName��������Ϣ������ָ�������г�ʼ����ͬʱ������ʹ��newInstance�������������ʵ����������ÿ���������ж��ṩ��һ����̬ע�����飬����mysql��

```
static {
    try {
        java.sql.DriverManager.registerDriver(new Driver());
    } catch (SQLException E) {
        throw new RuntimeException("Can't register driver!");
    }
}
```
������ʵ������һ�������࣬ͬʱע�ᵽDriverManager�����������ǵ���DriverManager��getConnection�������������£�

```
private static Connection getConnection(
       String url, java.util.Properties info, Class<?> caller) throws SQLException {
       /*
        * When callerCl is null, we should check the application's
        * (which is invoking this class indirectly)
        * classloader, so that the JDBC driver class outside rt.jar
        * can be loaded from here.
        */
       ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
       synchronized(DriverManager.class) {
           // synchronize loading of the correct classloader.
           if (callerCL == null) {
               callerCL = Thread.currentThread().getContextClassLoader();
           }
       }
 
       if(url == null) {
           throw new SQLException("The url cannot be null", "08001");
       }
 
       println("DriverManager.getConnection(\"" + url + "\")");
 
       // Walk through the loaded registeredDrivers attempting to make a connection.
       // Remember the first exception that gets raised so we can reraise it.
       SQLException reason = null;
 
       for(DriverInfo aDriver : registeredDrivers) {
           // If the caller does not have permission to load the driver then
           // skip it.
           if(isDriverAllowed(aDriver.driver, callerCL)) {
               try {
                   println("    trying " + aDriver.driver.getClass().getName());
                   Connection con = aDriver.driver.connect(url, info);
                   if (con != null) {
                       // Success!
                       println("getConnection returning " + aDriver.driver.getClass().getName());
                       return (con);
                   }
               } catch (SQLException ex) {
                   if (reason == null) {
                       reason = ex;
                   }
               }
 
           } else {
               println("    skipping: " + aDriver.getClass().getName());
           }
 
       }
 
       // if we got here nobody could connect.
       if (reason != null)    {
           println("getConnection failed: " + reason);
           throw reason;
       }
 
       println("getConnection: no suitable driver found for "+ url);
       throw new SQLException("No suitable driver found for "+ url, "08001");
   }
```
�˷�����Ҫ�Ǳ���֮ǰע���DriverInfo������url��Ϣȥÿ���������н������ӣ���Ȼÿ���������ж������urlƥ��У�飬�ɹ�֮�󷵻�Connection�������;��ʧ�ܵ����Ӳ���Ӱ�쳢���µ��������ӣ�������֮�����޷���ȡ���ӣ����׳��쳣��

###4.��չ
�������չ�µ�������Ҳ�ܼ򵥣�ֻ��Ҫ����·���´���META-INF/services/�ļ��У�ͬʱ�����洴��java.sql.Driver�ļ������ļ���д���������������ƣ���Ȼ������Ҫ�̳�java.sql.Driver�ӿ��ࣻ����ʵ�����ṩ��TestDriver��

##���л�ʵս
###1.׼���ӿ���

```
public interface Serialization {
 
    /**
     * ���л�
     * 
     * @param obj
     * @return
     */
    public byte[] serialize(Object obj) throws Exception;
 
    /**
     * �����л�
     * 
     * @param param
     * @param clazz
     * @return
     * @throws Exception
     */
    public <T> T deserialize(byte[] param, Class<T> clazz) throws Exception;
 
    /**
     * ���л�����
     * 
     * @return
     */
    public String getName();
 
}
```
###2.׼��ʵ����
�ֱ�׼��JsonSerialization��ProtobufSerialization

###3.�ӿ��ļ�
��META-INF/services/Ŀ¼�´����ļ�com.spi.serializer.Serialization���������£�

```
com.spi.serializer.JsonSerialization
com.spi.serializer.ProtobufSerialization
```
###4.�ṩManager��

```
public class SerializationManager {
 
    private static Map<String, Serialization> map = new HashMap<>();
 
    static {
        loadInitialSerializer();
    }
 
    private static void loadInitialSerializer() {
        ServiceLoader<Serialization> loadedSerializations = ServiceLoader.load(Serialization.class);
        Iterator<Serialization> iterator = loadedSerializations.iterator();
 
        try {
            while (iterator.hasNext()) {
                Serialization serialization = iterator.next();
                map.put(serialization.getName(), serialization);
            }
        } catch (Throwable t) {
            t.printStackTrace();
        }
    }
 
    public static Serialization getSerialization(String name) {
        return map.get(name);
    }
}
```
�ṩ����DriverManager��SerializationManager�࣬�ڼ������ʱ������������õ����л���ʽ���ṩһ��getSerialization�Ľ��췽������getConnection��

##�ܽ�
������JDBC����Ϊʵ�����ص��ʹ��ServiceLoader��ʽ�����ֽ��з�����ͬʱ�ṩ�����л��ļ�ʵս��dubboҲ�ṩ�����Ƶ�SPI��ʽ����������ExtensionLoader������java�ٷ��ṩ��ServiceLoader���ܸ�ǿ�󣬺�����������һ��dubbo��SPI��ʽ��Ȼ�����һ���Աȡ�

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][1]
[https://gitee.com/OutOfMemory/blog][2]


  [1]: https://github.com/ksfzhaohui/blog
  [2]: https://gitee.com/OutOfMemory/blog