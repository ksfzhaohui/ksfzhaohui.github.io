**前言**  
最近一个项目中使用了Hibernate的乐观锁，不巧的是出现了乐观锁最容易报的错：org.hibernate.StaleObjectStateException: Row was updated or deleted by another transaction  
下面将通过一个模拟的实例重现问题，并做相应的分析。

**乐观锁的作用**  
乐观锁的主要作用是为了解决事务并发带来的问题，相对于悲观锁而言，乐观锁机制采取了更加宽松的加锁机制；  
悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性，但随之而来的就是数据库性能的大量开销，特别是对长事务而言；  
乐观锁机制在一定程度上解决了这个问题；乐观锁，大多是基于数据版本（Version）记录机制实现；

乐观锁原理：读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一；  
此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据；  
当出现version过期时，会抛出如下错误：

```
public StaleObjectStateException(String persistentClass, Serializable identifier) {
    super("Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect)");
    this.entityName = persistentClass;
    this.identifier = identifier;
}
```

**本地重现**  
1.MYSQL建表SQL如下

```
CREATE TABLE `test_bean` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `version` int(11) NOT NULL,
  `status` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
 
INSERT INTO `test_bean` VALUES ('1', '1', '1');
```

其中的version字段，就是用来做乐观锁处理的

2.实体TestBean以及TestBean.hbm.xml

```
public class TestBean {
 
    private long id;
    private int version;
    private int status;
}
```

TestBean.hbm.xml如下：

```
<hibernate-mapping package="zh.maven.hibernate_version">
    <class name="TestBean" table="test_bean" optimistic-lock="version">
        <id name="id" column="id">
            <generator class="native" />
        </id>
        <version name="version" column="version" type="integer"></version>
        <property name="status" />
    </class>
</hibernate-mapping>
```

3.主要代码  
其中包括FirstService调用SecondService实现查询和更新，SecondService调用SecondDao实现查询和更新，代码如下：

```
public class FirstService {
 
    private SecondService secondService;
 
    public void modifyProcess() {
        TestBean testBean = secondService.get(1L);
        printLog(testBean);
        testBean.setStatus(4);
        secondService.update(testBean);
        printLog(testBean);
        sendNotify(testBean);
    }
 
    private void sendNotify(TestBean testBean) {
        testBean.setStatus(20);
        secondService.update(testBean);
        printLog(testBean);
    }
    ......
}
```

```
public class SecondService {
 
    private SecondDao secondDao;
 
    public TestBean get(long id) {
        return secondDao.get(id);
    }
 
    public void update(TestBean testBean) {
        secondDao.update(testBean);
    }
    ......
}
```

```
public class SecondDao extends HibernateDaoSupport {
 
    public TestBean get(long id) {
        return (TestBean) getHibernateTemplate().get(TestBean.class, 1L);
    }
 
    public void update(TestBean testBean) {
        getHibernateTemplate().update(testBean);
 
    }
}
```

3.核心配置spring-config-datasource.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">
 
 
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
        destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url"
            value="jdbc:mysql:///test?useUnicode=true&amp;characterEncoding=utf-8" />
        <property name="username" value="root" />
        <property name="password" value="root" />
        <property name="maxActive" value="100" />
        <property name="maxIdle" value="30" />
        <property name="maxWait" value="10000" />
    </bean>
    <bean id="sessionFactory"
        class="org.springframework.orm.hibernate3.LocalSessionFactoryBean"
        lazy-init="false">
        <property name="dataSource" ref="dataSource" />
        <property name="configLocation" value="classpath:hibernate.cfg.xml"></property>
        <property name="mappingLocations" value="TestBean.hbm.xml"></property>
    </bean>
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>
 
    <bean id="secondDao" class="zh.maven.hibernate_version.SecondDao">
        <property name="sessionFactory" ref="sessionFactory"></property>
    </bean>
    <bean id="secondService" class="zh.maven.hibernate_version.impl.service.SecondService">
        <property name="secondDao" ref="secondDao"></property>
    </bean>
    <bean id="firstService" class="zh.maven.hibernate_version.impl.service.FirstService">
        <property name="secondService" ref="secondService"></property>
    </bean>
 
    <aop:config>
        <aop:advisor pointcut="execution(* zh.maven..*Service*.*(..))"
            advice-ref="txAdvice" order="1" />
    </aop:config>
    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="update" propagation="REQUIRES_NEW"
                rollback-for="java.lang.Exception" />
            <tx:method name="*" propagation="SUPPORTS" rollback-for="java.lang.Exception" />
        </tx:attributes>
    </tx:advice
</beans>
```

配置了数据源，事务管理器，实体bean以及Spring AOP方式管理事务；为update方法配置了独立事务(REQUIRES_NEW)，其他方法配置了，其他方法配置了SUPPORTS;  
REQUIRES_NEW：每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行；  
SUPPORTS：如果上下文存在事务，则支持事务加入事务，如果没有事务，则使用非事务的方式执行。

4.测试  
测试类如下：

```
public class Test {
 
    public static void main(String[] args) throws Exception {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-config-datasource.xml");
        FirstService firstService = (FirstService) context.getBean("firstService");
        firstService.modifyProcess();
    }
}
```

运行结果如下：

```
Hibernate: select testbean0_.id as id0_0_, testbean0_.version as version0_0_, testbean0_.status as status0_0_ from test_bean testbean0_ where testbean0_.id=?
id:1,version=1,status=1
Hibernate: update test_bean set version=?, status=? where id=? and version=?
id:1,version=2,status=4
Hibernate: update test_bean set version=?, status=? where id=? and version=?
id:1,version=3,status=20
Hibernate: update test_bean set version=?, status=? where id=? and version=?
Exception in thread "main" org.springframework.orm.hibernate3.HibernateOptimisticLockingFailureException: Object of class [zh.maven.hibernate_version.TestBean] with identifier [1]: optimistic locking failed; nested exception is org.hibernate.StaleObjectStateException: Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect): [zh.maven.hibernate_version.TestBean#1]
    at org.springframework.orm.hibernate3.SessionFactoryUtils.convertHibernateAccessException(SessionFactoryUtils.java:669)
    at org.springframework.orm.hibernate3.SpringSessionSynchronization.beforeCommit(SpringSessionSynchronization.java:143)
    at org.springframework.transaction.support.TransactionSynchronizationUtils.triggerBeforeCommit(TransactionSynchronizationUtils.java:72)
    at org.springframework.transaction.support.AbstractPlatformTransactionManager.triggerBeforeCommit(AbstractPlatformTransactionManager.java:905)
    at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:715)
    at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:701)
    at org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:321)
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:116)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:171)
    at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:89)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:171)
    at org.springframework.aop.framework.Cglib2AopProxy$DynamicAdvisedInterceptor.intercept(Cglib2AopProxy.java:635)
    at zh.maven.hibernate_version.impl.service.FirstService$$EnhancerByCGLIB$$62556ba7.modifyProcess(<generated>)
    at zh.maven.hibernate_version.Test.main(Test.java:14)
```

代码中一共执行了3次sql操作，分别是：get和2次update，为什么会出现第四条update，并且因为第四条update导致了StaleObjectStateException；  
数据库结果：\[1,3,20\]，2次update，version：3，status：20，表示数据库中的数据是我们期望的。

**分析**  
这种情况一般都是由于配置的事务引发的，所以从Spring AOP事务入手；update配置了独立的事务，2次正常更新不会有什么影响；关键在于配置其他方法为SUPPORTS，  
但是SUPPORTS表示：如果上下文存在事务，则支持事务加入事务，如果没有事务，则使用非事务的方式执行，所以应该是不会提交事务的，直接深入源码查看；

Spring Aop方式的引入所有被过滤的方法都会经过拦截器TransactionInterceptor中，invoke方法中有如下代码：

```
TransactionInfo txInfo = createTransactionIfNecessary(txAttr, joinpointIdentification);
```

此方法用来创建事务信息，进入方法会发现通过PlatformTransactionManager来获取DefaultTransactionStatus对象，此对象里面有2个重要的属性分别是：

```
private final boolean newTransaction;
private final boolean newSynchronization;
```

getTransaction的部分代码如下：

```
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    ......
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
                definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        try {
                doBegin(transaction, definition);
            }
        ......
         
    }
    else {
            // Create "empty" transaction: no actual transaction, but potentially synchronization.
            boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
            return newTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
        }
}
```

PROPAGATION\_REQUIRED,PROPAGATION\_REQUIRES\_NEW,PROPAGATION\_NESTED这三种传播行为是会开启事务newTransaction=true  
其他传播方式包括这里配置的SUPPORTS分别是：newTransaction=false，newSynchronization=true，默认的transactionSynchronization=SYNCHRONIZATION_ALWAYS；  
Spring针对事务同步提供了类：TransactionSynchronization，其中有多个接口：beforeCommit，afterCommit等，分别事务提交之前，之后可以做相应的处理；  
这里主要关注beforeCommit方法，因为事务传播行为是SUPPORTS，所以不会doCommit，但是会triggerBeforeCommit；  
TransactionSynchronization具体实现类：SpringSessionSynchronization，其中对beforeCommit有如下实现：

```
public void beforeCommit(boolean readOnly) throws DataAccessException {
        if (!readOnly) {
            Session session = getCurrentSession();
            // Read-write transaction -> flush the Hibernate Session.
            // Further check: only flush when not FlushMode.NEVER/MANUAL.
            if (!session.getFlushMode().lessThan(FlushMode.COMMIT)) {
                try {
                    SessionFactoryUtils.logger.debug("Flushing Hibernate Session on transaction synchronization");
                    session.flush();
                }
                catch (HibernateException ex) {
                    if (this.jdbcExceptionTranslator != null && ex instanceof JDBCException) {
                        JDBCException jdbcEx = (JDBCException) ex;
                        throw this.jdbcExceptionTranslator.translate(
                                "Hibernate flushing: " + jdbcEx.getMessage(), jdbcEx.getSQL(), jdbcEx.getSQLException());
                    }
                    throw SessionFactoryUtils.convertHibernateAccessException(ex);
                }
            }
        }
    }
```

方法中首先获取当前的Session，然后对session进行flush操作，这里的session是get方法调用创建的session，和update操作创建的session没有关系；  
session会保存最近一次与数据库操作的信息记录的version=1，而TestBean从get到update始终都唯一对象，所以此时的TestBean已经是\[1,3,20\]，  
所以flush的时候会执行更新操作，更新语句如下：

```
update test_bean set version=2, status=20 where id=1 and version=3
```

set version=2因为保存的version=1，但是在update时候会加1，而此时数据库version为3，所以报StaleObjectStateException。

**解决**  
1.不给其他方法配置事务，直接移除

```
<tx:method name="*" propagation="SUPPORTS" rollback-for="java.lang.Exception" />
```

2.因为默认的transactionSynchronization=SYNCHRONIZATION\_ALWAYS，值为0，可以配置为SYNCHRONIZATION\_NEVER

```
<bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
        <property name="transactionSynchronization" value="2"></property>
</bean>
```

3.从代码入手从get对象到update一直在对持久化状态对象进行操作，与当前session关联，导致最后flush的时候dirtyCheck的时候通过了而导致update操作；  
所以可以对SecondService做如下修改：

```
public TestBean get(long id) {
    TestBean testBean= secondDao.get(id);
    return new TestBean(testBean.getId(), testBean.getVersion(), testBean.getStatus());
}
```

获取到的TestBean是一个临时的值，与当前session没有关联，与session关联的是TestBean testBean= secondDao.get(id)此对象后续没有变更，所以最后不会触发update操作。

**详细代码svn地址：**[http://code.taobao.org/svn/temp-pj/hibernate-version](http://code.taobao.org/svn/temp-pj/hibernate-version)