## 前言

Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成；接下来的几篇文章将重点分析ShardingSphere-JDBC，从数据分片，分布式主键，分布式事务，读写分离，弹性伸缩等几个方面来介绍。

## 简介

ShardingSphere-JDBC定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。整体架构图如下(来自官网)：

![shardingsphere-jdbc-brief.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed41222c0d39474ab5e72fe7e6aaeeee~tplv-k3u1fbpfcp-watermark.image)


ShardingSphere-JDBC包含了众多的功能模块包括数据分片，分布式主键，分布式事务，读写分离，弹性伸缩等等；作为一个数据库中间件最核心的功能当属数据分片了，ShardingSphere-JDBC提供了很多分库分表的策略和算法，接下来看看具体是如何使用这些策略的；



## 数据分片

作为一个开发者我们希望中间件可以帮我们屏蔽底层的细节，让我们在面对分库分表的场景下，可以像使用单库单表一样简单；当然ShardingSphere-JDBC不会让大家失望，引入了分片数据源、逻辑表等概念；

### 分片数据源和逻辑表

- 逻辑表：逻辑表是相对物理表来说的，通常做分表处理，某一张表会被分成多张表，比如订单表被拆分成10张表，分别是t_order_0到t_order_9，而对应的逻辑表就是`t_order`，对于开发者来说只需要使用逻辑表即可；
- 分片数据源：对于分库来说，通常会有多个库，或者说是多个数据源，所以这些数据源需要被统一管理起来，引入了分片数据源的概念，常见的`ShardingDataSource`

有了以上两个最基本的概念当然还不够，还需要分库分表策略算法帮助我们做路由处理；但是这两个概念可以让开发者有一种使用单库单表的感觉，就像下面这样一个简单的实例：

```java
DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
					new Properties());
Connection conn = dataSource.getConnection();
String sql = "select id,user_id,order_id from t_order where order_id = 103";
PreparedStatement preparedStatement = conn.prepareStatement(sql);
ResultSet set = preparedStatement.executeQuery();
```

以上根据真实数据源列表，分库分表策略生成了一个抽象数据源，可以简单理解就是`ShardingDataSource`；接下来的操作和我们使用jdbc操作正常的单库单表没有任何区别；

### 分片策略算法

ShardingSphere-JDBC在分片策略上分别引入了**分片算法**、**分片策略**两个概念，当然在分片的过程中**分片键**也是一个核心的概念；在此可以简单的理解`分片策略 = 分片算法 + 分片键`；至于为什么要这么设计，应该是ShardingSphere-JDBC考虑更多的灵活性，把分片算法单独抽象出来，方便开发者扩展；

#### 分片算法

提供了抽象分片算法类：`ShardingAlgorithm`，根据类型又分为：精确分片算法、区间分片算法、复合分片算法以及Hint分片算法；

- 精确分片算法：对应`PreciseShardingAlgorithm`类，主要用于处理 `=` 和 `IN`的分片；
- 区间分片算法：对应`RangeShardingAlgorithm`类，主要用于处理 `BETWEEN AND`, `>`, `<`, `>=`, `<=` 分片；
- 复合分片算法：对应`ComplexKeysShardingAlgorithm`类，用于处理使用多键作为分片键进行分片的场景；
- Hint分片算法：对应`HintShardingAlgorithm`类，用于处理使用 `Hint` 行分片的场景；

以上所有的算法类都是接口类，具体实现交给开发者自己；

#### 分片策略

分片策略基本和上面的分片算法对应，包括：标准分片策略、复合分片策略、Hint分片策略、内联分片策略、不分片策略；

- 标准分片策略：对应`StandardShardingStrategy`类，提供`PreciseShardingAlgorithm`和`RangeShardingAlgorithm`两个分片算法，`PreciseShardingAlgorithm`是必须的，`RangeShardingAlgorithm`可选的；

  ```java
  public final class StandardShardingStrategy implements ShardingStrategy {
      private final String shardingColumn;
      private final PreciseShardingAlgorithm preciseShardingAlgorithm;
      private final RangeShardingAlgorithm rangeShardingAlgorithm;
  }
  ```

- 复合分片策略：对应`ComplexShardingStrategy`类，提供`ComplexKeysShardingAlgorithm`分片算法；

  ```java
  public final class ComplexShardingStrategy implements ShardingStrategy {
      @Getter
      private final Collection<String> shardingColumns;
      private final ComplexKeysShardingAlgorithm shardingAlgorithm;
  }
  ```

  可以发现支持多个分片键；

- Hint分片策略：对应`HintShardingStrategy`类，通过 Hint 指定分片值而非从 SQL 中提取分片值的方式进行分片的策略；提供`HintShardingAlgorithm`分片算法；

  ```java
  public final class HintShardingStrategy implements ShardingStrategy {
      @Getter
      private final Collection<String> shardingColumns;
      private final HintShardingAlgorithm shardingAlgorithm;
  }
  ```

- 内联分片策略：对应`InlineShardingStrategy`类，没有提供分片算法，路由规则通过表达式来实现；

- 不分片策略：对应`NoneShardingStrategy`类，不分片策略；

#### 分片策略配置类

在使用中我们并没有直接使用上面的分片策略类，ShardingSphere-JDBC分别提供了对应策略的配置类包括：

- `StandardShardingStrategyConfiguration`
- `ComplexShardingStrategyConfiguration`
- `HintShardingStrategyConfiguration`
- `InlineShardingStrategyConfiguration`
- `NoneShardingStrategyConfiguration`

### 实战

有了以上相关基础概念，接下来针对每种分片策略做一个简单的实战，在实战前首先准备好库和表；

#### 准备

分别准备两个库：`ds0`、`ds1`；然后每个库分别包含两个表：`t_order0`，`t_order1`；

```sql
CREATE TABLE `t_order0` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL,
  `order_id` bigint(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

#### 准备真实数据源

我们这里有两个数据源，这里都使用java代码的方式来配置：

```java
// 配置真实数据源
Map<String, DataSource> dataSourceMap = new HashMap<>();

// 配置第一个数据源
BasicDataSource dataSource1 = new BasicDataSource();
dataSource1.setDriverClassName("com.mysql.jdbc.Driver");
dataSource1.setUrl("jdbc:mysql://localhost:3306/ds0");
dataSource1.setUsername("root");
dataSource1.setPassword("root");
dataSourceMap.put("ds0", dataSource1);

// 配置第二个数据源
BasicDataSource dataSource2 = new BasicDataSource();
dataSource2.setDriverClassName("com.mysql.jdbc.Driver");
dataSource2.setUrl("jdbc:mysql://localhost:3306/ds1");
dataSource2.setUsername("root");
dataSource2.setPassword("root");
dataSourceMap.put("ds1", dataSource2);
```

这里配置的两个数据源都是普通的数据源，最后会把dataSourceMap交给`ShardingDataSourceFactory`管理；

#### 表规则配置

表规则配置类`TableRuleConfiguration`，包含了五个要素：逻辑表、真实数据节点、数据库分片策略、数据表分片策略、分布式主键生成策略；

```java
TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration("t_order", "ds${0..1}.t_order${0..1}");

orderTableRuleConfig.setDatabaseShardingStrategyConfig(
				new StandardShardingStrategyConfiguration("user_id", new MyPreciseSharding()));
orderTableRuleConfig.setTableShardingStrategyConfig(
				new StandardShardingStrategyConfiguration("order_id", new MyPreciseSharding()));

orderTableRuleConfig.setKeyGeneratorConfig(new KeyGeneratorConfiguration("SNOWFLAKE", "id"));
```

- 逻辑表：这里配置的逻辑表就是t_order，对应的物理表有t_order0，t_order1；

- 真实数据节点：这里使用行表达式进行配置的，简化了配置；上面的配置就相当于配置了：

  ```xml
  db0
    ├── t_order0 
    └── t_order1 
  db1
    ├── t_order0 
    └── t_order1
  ```

- 数据库分片策略：这里的库分片策略就是上面介绍的五种类型，这里使用的`StandardShardingStrategyConfiguration`，需要指定**分片键**和**分片算法**，这里使用的是**精确分片算法**；

  ```java
  public class MyPreciseSharding implements PreciseShardingAlgorithm<Integer> {
  
  	@Override
  	public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Integer> shardingValue) {
  		Integer index = shardingValue.getValue() % 2;
  		for (String target : availableTargetNames) {
  			if (target.endsWith(index + "")) {
  				return target;
  			}
  		}
  		return null;
  	}
  }
  ```

  这里的shardingValue就是user_id对应的真实值，每次和2取余；availableTargetNames可选择就是{ds0，ds1}；看余数和哪个库能匹配上就表示路由到哪个库；

- 数据表分片策略：指定的**分片键(order_id)**和分库策略不一致，其他都一样；

- 分布式主键生成策略：ShardingSphere-JDBC提供了多种分布式主键生成策略，后面详细介绍，这里使用雪花算法；

#### 配置分片规则

配置分片规则`ShardingRuleConfiguration`，包括多种配置规则：表规则配置、绑定表配置、广播表配置、默认数据源名称、默认数据库分片策略、默认表分片策略、默认主键生成策略、主从规则配置、加密规则配置；

- 表规则配置
  **tableRuleConfigs**：也就是上面配置的库分片策略和表分片策略，也是最常用的配置；

- 绑定表配置
  **bindingTableGroups**：指分⽚规则⼀致的主表和⼦表；绑定表之间的多表关联查询不会出现笛卡尔积关联，关联查询效率将⼤⼤提升；

- 广播表配置
  **broadcastTables**：所有的分⽚数据源中都存在的表，表结构和表中的数据在每个数据库中均完全⼀致。适⽤于数据量不⼤且需要与海量数据的表进⾏关联查询的场景；

- 默认数据源名称
  **defaultDataSourceName**：未配置分片的表将通过默认数据源定位；

- 默认数据库分片策略
  defaultDatabaseShardingStrategyConfig：表规则配置可以设置数据库分片策略，如果没有配置可以在这里面配置默认的；

- 默认表分片策略
  **defaultTableShardingStrategyConfig**：表规则配置可以设置表分片策略，如果没有配置可以在这里面配置默认的；

- 默认主键生成策略
  **defaultKeyGeneratorConfig**：表规则配置可以设置主键生成策略，如果没有配置可以在这里面配置默认的；内置UUID、SNOWFLAKE生成器；

- 主从规则配置
  **masterSlaveRuleConfigs**：用来实现读写分离的，可配置一个主表多个从表，读面对多个从库可以配置负载均衡策略；

- 加密规则配置
  **encryptRuleConfig**：提供了对某些敏感数据进行加密的功能，提供了⼀套完整、安全、透明化、低改造成本的数据加密整合解决⽅案；

#### 数据插入

以上准备好，就可以操作数据库了，这里执行插入操作：

```java
DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
				new Properties());
Connection conn = dataSource.getConnection();
String sql = "insert into t_order (user_id,order_id) values (?,?)";
PreparedStatement preparedStatement = conn.prepareStatement(sql);
for (int i = 1; i <= 10; i++) {
	preparedStatement.setInt(1, i);
	preparedStatement.setInt(2, 100 + i);
	preparedStatement.executeUpdate();
}
```

通过以上配置的真实数据源、分片规则以及属性文件创建分片数据源`ShardingDataSource`；接下来就可以像使用单库单表一样操作分库分表了，sql中可以直接使用逻辑表，分片算法会根据具体的值就行路由处理；

经过路由最终：奇数入ds1.t_order1，偶数入ds0.t_order0；以上使用了最常见的精确分片算法，下面继续看一下其他几种分片算法；

#### 分片算法

上面的介绍的精确分片算法中，通过`PreciseShardingValue`来获取当前分片键值，ShardingSphere-JDBC针对每种分片算法都提供了相应的`ShardingValue`，具体包括：

- PreciseShardingValue
- RangeShardingValue
- ComplexKeysShardingValue
- HintShardingValue

##### 区间分片算法

用在区间查询的时候，比如下面的查询SQL：

```sql
select * from t_order where order_id>2 and order_id<9
```

以上两个区间值2、9会直接保存到`RangeShardingValue`中，这里没有指定user_id用来做库路由，所以会访问两个库；

```java
public class MyRangeSharding implements RangeShardingAlgorithm<Integer> {

	@Override
	public Collection<String> doSharding(Collection<String> availableTargetNames,
			RangeShardingValue<Integer> shardingValue) {
		Collection<String> result = new LinkedHashSet<>();
		Range<Integer> range = shardingValue.getValueRange();

		// 区间开始和结束值
		int lower = range.lowerEndpoint();
		int upper = range.upperEndpoint();

		for (int i = lower; i <= upper; i++) {
			Integer index = i % 2;
			for (String target : availableTargetNames) {
				if (target.endsWith(index + "")) {
					result.add(target);
				}
			}
		}
		return result;
	}

}
```

可以发现会检查区间开始和结束中的每个值和2取余，是否都能和真实的表匹配；

##### 复合分片算法

可以同时使用多个分片键，比如可以同时使用user_id和order_id作为分片键；

```java
orderTableRuleConfig.setDatabaseShardingStrategyConfig(
				new ComplexShardingStrategyConfiguration("order_id,user_id", new MyComplexKeySharding()));
orderTableRuleConfig.setTableShardingStrategyConfig(
				new ComplexShardingStrategyConfiguration("order_id,user_id", new MyComplexKeySharding()));
```

如上在配置分库分表策略时，指定了两个分片键，用逗号隔开；分片算法如下：

```java
public class MyComplexKeySharding implements ComplexKeysShardingAlgorithm<Integer> {

	@Override
	public Collection<String> doSharding(Collection<String> availableTargetNames,
			ComplexKeysShardingValue<Integer> shardingValue) {
		Map<String, Collection<Integer>> map = shardingValue.getColumnNameAndShardingValuesMap();

		Collection<Integer> userMap = map.get("user_id");
		Collection<Integer> orderMap = map.get("order_id");

		List<String> result = new ArrayList<>();
		// user_id，order_id分片键进行分表
		for (Integer userId : userMap) {
			for (Integer orderId : orderMap) {
				int suffix = (userId+orderId) % 2;
				for (String s : availableTargetNames) {
					if (s.endsWith(suffix+"")) {
						result.add(s);
					}
				}
			}
		}
		return result;
	}
}
```

##### Hint分片算法

在一些应用场景中，分片条件并不存在于 SQL，而存在于外部业务逻辑；可以通过编程的方式向 `HintManager` 中添加分片条件，该分片条件仅在当前线程内生效；

```java
// 设置库表分片策略
orderTableRuleConfig.setDatabaseShardingStrategyConfig(new HintShardingStrategyConfiguration(new 		MyHintSharding()));
orderTableRuleConfig.setTableShardingStrategyConfig(new HintShardingStrategyConfiguration(new MyHintSharding()));

// 手动设置分片条件
int hitKey1[] = { 2020, 2021, 2022, 2023, 2024 };
int hitKey2[] = { 3020, 3021, 3022, 3023, 3024 };
DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
				new Properties());
Connection conn = dataSource.getConnection();
for (int i = 1; i <= 5; i++) {
	final int index = i;
	new Thread(new Runnable() {
	@Override
	public void run() {
			try {
				HintManager hintManager = HintManager.getInstance();
				String sql = "insert into t_order (user_id,order_id) values (?,?)";
				PreparedStatement preparedStatement = conn.prepareStatement(sql);
                // 分别添加库和表分片条件
				hintManager.addDatabaseShardingValue("t_order", hitKey1[index - 1]);
				hintManager.addTableShardingValue("t_order", hitKey2[index - 1]);
						
				preparedStatement.setInt(1, index);
				preparedStatement.setInt(2, 100 + index);
				preparedStatement.executeUpdate();
				} catch (SQLException e) {
					e.printStackTrace();
				}
			}
		}).start();
}
```

以上实例中，手动设置了分片条件，分片算法如下所示：

```java
public class MyHintSharding implements HintShardingAlgorithm<Integer> {

	@Override
	public Collection<String> doSharding(Collection<String> availableTargetNames,
			HintShardingValue<Integer> shardingValue) {
		List<String> shardingResult = new ArrayList<>();
		for (String targetName : availableTargetNames) {
			String suffix = targetName.substring(targetName.length() - 1);
			Collection<Integer> values = shardingValue.getValues();
			for (int value : values) {
				if (value % 2 == Integer.parseInt(suffix)) {
					shardingResult.add(targetName);
				}
			}
		}
		return shardingResult;
	}
}

```

##### 不分片

配置`NoneShardingStrategyConfiguration`即可：

```java
orderTableRuleConfig.setDatabaseShardingStrategyConfig(new NoneShardingStrategyConfiguration());
orderTableRuleConfig.setTableShardingStrategyConfig(new NoneShardingStrategyConfiguration());
```

这样数据会插入每个库每张表，可以理解为`广播表`

## 分布式主键

面对多个数据库表需要有唯一的主键，引入了分布式主键功能，内置的主键生成器包括：UUID、SNOWFLAKE；

### UUID

直接使用UUID.randomUUID()生成，主键没有任何规则；对应的主键生成类：`UUIDShardingKeyGenerator`；

### SNOWFLAKE

实现类：`SnowflakeShardingKeyGenerator`；使⽤雪花算法⽣成的主键，⼆进制表⽰形式包含 4 部分，从⾼位到低位分表为：1bit 符号位、41bit 时间戳位、10bit ⼯作进程位以及 12bit 序列号位；来自官网的图片：

![image-20210415191555431.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c3235719c2e4f40a26bdfccab09dfe6~tplv-k3u1fbpfcp-watermark.image)



### 扩展

实现接口：`ShardingKeyGenerator`，实现自己的主键生成器；

```java
public interface ShardingKeyGenerator extends TypeBasedSPI {
    Comparable<?> generateKey();
}
```

### 实战

使用也很简单，直接使用`KeyGeneratorConfiguration`即可，配置对应的算法类型和字段名称：

```java
orderTableRuleConfig.setKeyGeneratorConfig(new KeyGeneratorConfiguration("SNOWFLAKE", "id"));
```

这里使用雪花算法生成器，对应生成的字段是id；结果如下：

```sql
mysql> select * from t_order0;
+--------------------+---------+----------+
| id                 | user_id | order_id |
+--------------------+---------+----------+
| 589535589984894976 |       0 |        0 |
| 589535590504988672 |       2 |        2 |
| 589535590718898176 |       4 |        4 |
+--------------------+---------+----------+
```

## 分布式事务

ShardingSphere-JDBC使用分布式事务和使用本地事务没什么区别，提供了透明化的分布式事务；支持的事务类型包括：本地事务、XA事务和柔性事务，默认是本地事务；

```java
public enum TransactionType {
    LOCAL, XA, BASE
}
```

### 依赖

根据具体使用XA事务还是柔性事务，需要引入不同的模块；

```xml
<dependency>
	<groupId>org.apache.shardingsphere</groupId>
	<artifactId>sharding-transaction-xa-core</artifactId>
</dependency>

<dependency>
	<groupId>org.apache.shardingsphere</groupId>
	<artifactId>shardingsphere-transaction-base-seata-at</artifactId>
</dependency>
```

### 实现

ShardingSphere-JDBC提供了分布式事务管理器`ShardingTransactionManager`，实现包括：

- XAShardingTransactionManager：基于 XA 的分布式事务管理器；
- SeataATShardingTransactionManager：基于 Seata 的分布式事务管理器；

 XA 的分布式事务管理器具体实现包括：Atomikos、Narayana、Bitronix；默认是Atomikos；

### 实战

默认的事务类型是TransactionType.LOCAL，ShardingSphere-JDBC天生面向多数据源，本地模式其实是循环提交每个数据源的事务，不能保证数据的一致性，所以需要使用分布式事务，具体使用也很简单：

```java
//改变事务类型为XA
TransactionTypeHolder.set(TransactionType.XA);
DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
				new Properties());
Connection conn = dataSource.getConnection();
try {
	//关闭自动提交
	conn.setAutoCommit(false);
			
	String sql = "insert into t_order (user_id,order_id) values (?,?)";
	PreparedStatement preparedStatement = conn.prepareStatement(sql);
	for (int i = 1; i <= 5; i++) {
		preparedStatement.setInt(1, i - 1);
		preparedStatement.setInt(2, i - 1);
		preparedStatement.executeUpdate();
	}
	//事务提交
	conn.commit();
} catch (Exception e) {
	e.printStackTrace();
	//事务回滚
	conn.rollback();
}
```

可以发现使用起来还是很简单的，ShardingSphere-JDBC会根据当前的事务类型，在提交的时候判断是走本地事务提交，还是使用分布式事务管理器`ShardingTransactionManager`进行提交；

## 读写分离

对于同一时刻有大量并发读操作和较少写操作类型的应用系统来说，将数据库拆分为主库和从库，主库负责处理事务性的增删改操作，从库负责处理查询操作，能够有效的避免由数据更新导致的行锁，使得整个系统的查询性能得到极大的改善。

### 主从配置

在上面章节介绍分片规则的时候，其中有说到主从规则配置，其目的就是用来实现读写分离的，核心配置类：`MasterSlaveRuleConfiguration`：

```java
public final class MasterSlaveRuleConfiguration implements RuleConfiguration {
    private final String name;
    private final String masterDataSourceName;
    private final List<String> slaveDataSourceNames;
    private final LoadBalanceStrategyConfiguration loadBalanceStrategyConfiguration;
}
```

- name：配置名称，当前使用的4.1.0版本，这里必须是主库的名称；
- masterDataSourceName：主库数据源名称；
- slaveDataSourceNames：从库数据源列表，可以配置一主多从；
- LoadBalanceStrategyConfiguration：面对多个从库，读取的时候会通过负载算法进行选择；

主从负载算法类：`MasterSlaveLoadBalanceAlgorithm`，实现类包括：随机和循环；

- ROUND_ROBIN：实现类`RoundRobinMasterSlaveLoadBalanceAlgorithm`
- RANDOM：实现类`RandomMasterSlaveLoadBalanceAlgorithm`

### 实战

分别给ds0和ds1准备从库：ds01和ds11，分别配置主从同步；读写分离配置如下：

```java
List<String> slaveDataSourceNames0 = new ArrayList<String>();
slaveDataSourceNames0.add("ds01");
MasterSlaveRuleConfiguration masterSlaveRuleConfiguration0 = new MasterSlaveRuleConfiguration("ds0", "ds0",
				slaveDataSourceNames0);
shardingRuleConfig.getMasterSlaveRuleConfigs().add(masterSlaveRuleConfiguration0);
		
List<String> slaveDataSourceNames1 = new ArrayList<String>();
slaveDataSourceNames1.add("ds11");
MasterSlaveRuleConfiguration masterSlaveRuleConfiguration1 = new MasterSlaveRuleConfiguration("ds1", "ds1",
				slaveDataSourceNames1);
shardingRuleConfig.getMasterSlaveRuleConfigs().add(masterSlaveRuleConfiguration1);
```

这样在执行查询操作的时候会自动路由到从库，实现读写分离；

## 总结

本文重点介绍了ShardingSphere-JDBC的数据分片功能，这也是所有数据库中间件的核心功能；当然分布式主键、分布式事务、读写分离等功能也是必不可少的；同时ShardingSphere还引入了`弹性伸缩`的功能，这是一个非常亮眼的功能，因为数据库分片本身是有状态的，所以我们在项目启动之初都固定了多少库多少表，然后通过分片算法路由到各个库表，但是业务的发展往往超乎我们的预期，这时候如果想扩表扩库会很麻烦，目前看ShardingSphere官网`弹性伸缩`处于**alpha**开发阶段，非常期待此功能。

## 参考

https://shardingsphere.apache.org/document/current/cn/overview/

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。
