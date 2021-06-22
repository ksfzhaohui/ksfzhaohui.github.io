## 前言

上文[ShardingSphere-JDBC分片解析引擎](https://juejin.cn/post/6954171863425613854)中介绍了分片流程中的解析引擎，重点介绍了解析引擎的核心组件`ANTLR`；本文继续介绍分片流程中的路由引擎，路由引擎可以说是整个分片流程的核心模块，用户自定义的分片算法都在路由引擎中执行；



## 开启日志

为了更加清晰详细的查看路由日志，开启`SQL_SHOW`功能：

- 引入log4j相关jar，以及log4j.xml配置文件

- 配置`SQL_SHOW`属性为开启：

  ```java
  Properties prop = new Properties();
  prop.put(ConfigurationPropertyKey.SQL_SHOW.getKey(), true);
  ```

## 路由装饰器

路由引擎的外层包装了一层路由装饰器`RouteDecorator`，为什么要这么设计是因为除了我们正常走路由算法的路由引擎，`ShardingSphere-JDBC`还提供了读写分离功能，这其实在一定程度上来讲也是路由，而且这两种路由方式是可以叠加的；所有这里提供了一层抽象，实现类包括：

- MasterSlaveRouteDecorator：读写分离路由装饰器；
- ShardingRouteDecorator：分片路由装饰器，内部包含了各种路由引擎；

装饰器可以叠加，所以提供了优先级功能`OrderAware`，同时每个装饰器都有对应的规则，大致如下所示：

| 装饰器-RouteDecorator     | 配置-Configuration           | 规则-BaseRule   | 优先级-Order |
| ------------------------- | ---------------------------- | --------------- | ------------ |
| MasterSlaveRouteDecorator | MasterSlaveRuleConfiguration | MasterSlaveRule | 10           |
| ShardingRouteDecorator    | ShardingRuleConfiguration    | ShardingRule    | 0            |

根据优先级可以知道首先执行`ShardingRouteDecorator`，有了路由结果再执行`MasterSlaveRouteDecorator`；部分启动类代码在`DataNodeRouter`中如下所示：

```java
private final Map<BaseRule, RouteDecorator> decorators = new LinkedHashMap<>();
private RouteContext executeRoute(final String sql, final List<Object> parameters, final boolean useCache) {
      RouteContext result = createRouteContext(sql, parameters, useCache);
      for (Entry<BaseRule, RouteDecorator> entry : decorators.entrySet()) {
          result = entry.getValue().decorate(result, metaData, entry.getKey(), properties);
      }
      return result;
}
```

`decorators`会根据用户的配置来决定是否会启动对应的装饰器，可以参考上面的表格；下面按照优先级分别介绍两种装饰器；



## 分片路由装饰器

经过解析引擎获取到了`SQLStatement`，想要做分片路由除了此参数还需要另外一个重要参数分片路由规则`ShardingRule` ；有了这两个核心参数分片路由大致可以分为以下几步：

- 获取分片条件`ShardingConditions`
- 获取具体分片引擎`ShardingRouteEngine`
- 执行路由处理，获取路由结果

在详细介绍每一步之前，首先介绍以下几个核心参数`RouteContext`、`ShardingRule`；

### 核心参数

重点看一下`RouteContext`、`ShardingRule`这两个核心参数；

#### RouteContext

路由上下文参数，主要包含如下几个参数：

```java
public final class RouteContext {
    private final SQLStatementContext sqlStatementContext;
    private final List<Object> parameters;
    private final RouteResult routeResult;
}
```

- sqlStatementContext：解析引擎获取的`SQLStatement`；
- parameters：`PreparedStatement`中设置的参数，如执行insert操作`setXxx`代替`?`；
- routeResult：路由之后用来存放路由结果；

#### ShardingRule

分片规则，主要参数如下，这个其实和`ShardingRuleConfiguration`大同小异，只是重新做了一个包装；

```java
public class ShardingRule implements BaseRule {
    private final ShardingRuleConfiguration ruleConfiguration;
    private final ShardingDataSourceNames shardingDataSourceNames;
    private final Collection<TableRule> tableRules;
    private final Collection<BindingTableRule> bindingTableRules;
    private final Collection<String> broadcastTables;
    private final ShardingStrategy defaultDatabaseShardingStrategy;
    private final ShardingStrategy defaultTableShardingStrategy;
    private final ShardingKeyGenerator defaultShardingKeyGenerator;
    private final Collection<MasterSlaveRule> masterSlaveRules;
    private final EncryptRule encryptRule;
```

- ruleConfiguration：路由规则配置，可以理解为是`ShardingRule`的原始文件；
- shardingDataSourceNames：分片的数据源名称；
- tableRules：表路由规则，对应了用户配置的`TableRuleConfiguration`；
- bindingTableRules：绑定表配置，分⽚规则⼀致的主表和⼦表；
- broadcastTables：广播表配置，所有的分⽚数据源中都存在的表；
- defaultDatabaseShardingStrategy：默认库分片策略；
- defaultTableShardingStrategy：默认表分片策略；
- defaultShardingKeyGenerator：默认主键生成策略；
- masterSlaveRules：主从规则配置，用来实现读写分离的，可配置一个主表多个从表；
- encryptRule：加密规则配置，提供了对某些敏感数据进行加密的功能；

### 获取分片条件

在获取具体路由引擎和执行路由操作之前，我们需要获取分片的条件，常见的分片条件主要在Insert语句和Where语句后面；部分获取分片条件的源码如下：

```java
    private ShardingConditions getShardingConditions(final List<Object> parameters, 
                                                     final SQLStatementContext sqlStatementContext, final SchemaMetaData schemaMetaData, final ShardingRule shardingRule) {
        if (sqlStatementContext.getSqlStatement() instanceof DMLStatement) {
            if (sqlStatementContext instanceof InsertStatementContext) {
                return new ShardingConditions(new InsertClauseShardingConditionEngine(shardingRule).createShardingConditions((InsertStatementContext) sqlStatementContext, parameters));
            }
            return new ShardingConditions(new WhereClauseShardingConditionEngine(shardingRule, schemaMetaData).createShardingConditions(sqlStatementContext, parameters));
        }
        return new ShardingConditions(Collections.emptyList());
    }
```

首先会判断是不是`DMLStatement`类型，也是最常见的SQL类型比如：增删改查；接下来判断是否是Insert语句，分别使用不同的分片条件生成引擎：

- InsertClauseShardingConditionEngine：处理Insert语句分片条件；
- WhereClauseShardingConditionEngine：处理非Insert语句Where语句的分片条件；

#### InsertClauseShardingConditionEngine

Insert语句中包含分片条件主要有两个地方：

- Insert语句中指定的字段名称，当然是否是分片条件，还需要检测`ShardingRule`中的`tableRules`是否配置了相关字段作为分片键，看一条简单的Insert语句：

  ```sql
  insert into t_order (user_id,order_id) values (?,?)
  ```

  这一步其实就是检测user_id和order_id是否在`ShardingRule`中配置成分片键；

- 当然除了上面显示指定的字段，还有无需显示指定的主键，如果配置了主键生成策略，同样需要检测`ShardingRule`中的`tableRules`是否配置了相关字段作为分片键；

  ```java
  tableRuleConfig.setKeyGeneratorConfig(new KeyGeneratorConfiguration("SNOWFLAKE", "id"));
  ```

生成的结果就是`ShardingConditions`，内部包含多个`ShardingCondition`：

```java
public final class ShardingConditions {
    private final List<ShardingCondition> conditions;
}

public class ShardingCondition {
    private final List<RouteValue> routeValues = new LinkedList<>();
}
```

这里的`RouteValue`实现包括`ListRouteValue`、`RangeRouteValue`、`AlwaysFalseRouteValue`：

- ListRouteValue：可以理解分片键对应的是一个具体的值，可以是单个也可以多个；
- RangeRouteValue：分片键对应的是一个区间值；
- AlwaysFalseRouteValue：总是失败的路由值；

#### WhereClauseShardingConditionEngine

常见Select语句，Where语句后面可以包含多个条件，每个条件同样需要检测`ShardingRule`中的`tableRules`是否配置了相关字段作为分片键；稍有不同的地方是，Where条件需要做合并处理，比如：

```sql
String sql = "select user_id,order_id from t_order where order_id = 101 and order_id = 101";
String sql = "select user_id,order_id from t_order where order_id = 101 and order_id in(101)";
```

order_id出现多个值一样会进行合并处理，这里会合并成一个`order_id = 101`，如果这里两个值不一样比如：

```sql
String sql = "select user_id,order_id from t_order where order_id = 101 and order_id = 102";
```

会返回一个`AlwaysFalseRouteValue`，表示这个条件不可能成立；

### 获取路由引擎

`ShardingSphere-JDBC`根据不同的`SQLStatement`提供了10种路由引擎，下面分别介绍，首先看一下大致的流程图；

#### 流程图


![流程图.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81a422844ae649d1baaed62a67542601~tplv-k3u1fbpfcp-watermark.image)

流程图大致如上所示，具体查看`ShardingRouteEngineFactory`即可；下面详细介绍每个路由引擎；

#### 路由引擎

#### ShardingDatabaseBroadcastRoutingEngine

全库路由引擎：用于处理对数据库的操作，包括用于库设置的 SET 类型的数据库管理命令，以及 TCL 这样的事务控制语句。 在这种情况下，会根据逻辑库的名字遍历所有符合名字匹配的真实库，并在真实库中执行该命令；

##### 1.属于DALStatement

数据库访问层，常见的命令包括：set，reset，show databases；

```sql
show databases;
```

以上sql会全库路由，路由sql如下所示：

```sql
Actual SQL: ds0 ::: show databases;
Actual SQL: ds1 ::: show databases;
```

##### 2.逻辑表都属于广播表

```sql
insert into t_config (k,v) values (?,?)
```

`t_config`配置的是一张广播表，执行`insert`操作会将数据插入所有库中；当然前提需要配置广播表：

```java
Collection<String> broadcastTables = new LinkedList<>();
broadcastTables.add("t_config");
shardingRuleConfig.setBroadcastTables(broadcastTables);
```

路由日志如下：

```sql
Actual SQL: ds0 ::: insert into t_config (k,v) values (?, ?) ::: [aa1, 1111]
Actual SQL: ds1 ::: insert into t_config (k,v) values (?, ?) ::: [aa1, 1111]
```

##### 3.属于TCLStatement

事务控制语言，包括设置保存点，回滚等；

```sql
SET autocommit=0
```

路由日志如下：

```sql
Actual SQL: ds0 ::: SET autocommit=0;
Actual SQL: ds1 ::: SET autocommit=0;
```

#### ShardingTableBroadcastRoutingEngine

全库表路由用于处理对数据库中与其逻辑表相关的所有真实表的操作，主要包括不带分片键的 DQL 和 DML，以及 DDL 等；

##### 1.属于DDLStatement

数据库定义语言，包括创建、修改、删除表等；

```sql
ALTER  TABLE t_order MODIFY  COLUMN user_id  BIGINT(50) NOT NULL;
```

日志如下所示：

```sql
Actual SQL: ds0 ::: ALTER  TABLE t_order0 MODIFY  COLUMN user_id  BIGINT(50) NOT NULL;
Actual SQL: ds0 ::: ALTER  TABLE t_order1 MODIFY  COLUMN user_id  BIGINT(50) NOT NULL;
Actual SQL: ds1 ::: ALTER  TABLE t_order0 MODIFY  COLUMN user_id  BIGINT(50) NOT NULL;
Actual SQL: ds1 ::: ALTER  TABLE t_order1 MODIFY  COLUMN user_id  BIGINT(50) NOT NULL;
```

##### 2.属于DCLStatement

数据库控制语言，包括授权，角色控制等；grant，deny等命令

```sql
grant select on ds.t_order to root@'%'
```

给用户root授权select权限，这里需要指定唯一的表名，不能使用*代替；

```sql
Actual SQL: ds0 ::: grant select on t_order0 to root@'%'
Actual SQL: ds0 ::: grant select on t_order1 to root@'%'
Actual SQL: ds1 ::: grant select on t_order0 to root@'%'
Actual SQL: ds1 ::: grant select on t_order1 to root@'%'
```

#### ShardingIgnoreRoutingEngine

阻断路由用于屏蔽 SQL 对数据库的操作；

##### 1.DALStatement

阻断路由主要针对`DALStatement`下面的use命令；

```sql
use ds0
```

#### ShardingDefaultDatabaseRoutingEngine

默认数据库路由，需要配置默认数据源名称，包含以下几种情况：

##### 1.属于DALStatement

```sql
show create table t_order1
```

**注意点**：这里表是真实表，不能配置`TableRuleConfiguration`，需要配置默认数据源名称；

```java
shardingRuleConfig.setDefaultDataSourceName("ds0");
```

日志如下：

```
Actual SQL: ds0 ::: show create table t_order1
```

##### 2.逻辑表都属于默认数据源

```sql
select user_id,order_id from t_order0 where user_id = 102
```

**注意点**：这里表是真实表，不能配置`TableRuleConfiguration`，需要配置默认数据源名称，不能配置为广播表；

```
Actual SQL: ds0 ::: select user_id,order_id from t_order0 where user_id = 102
```

##### 3.属于DMLStatement

```sql
select 2+2
```

此`DMLStatement`没有表名，并且需要指定默认数据源名称；

```sql
Actual SQL: ds0 ::: select 2+2
```

#### ShardingUnicastRoutingEngine

单播路由用于获取某一真实表信息的场景，它仅需要从任意库中的任意真实表中获取数据即可；

##### 1.属于DALStatement

```
desc t_order
```

日志如下：

```
Actual SQL: ds1 ::: desc t_order0
```

##### 2.逻辑表都属于广播表

广播表分片数据源中都存在的表，一般是字典表：

```sql
select * from t_config
```

需要配置广播表：

```java
Collection<String> broadcastTables = new LinkedList<>();
broadcastTables.add("t_config");
shardingRuleConfig.setBroadcastTables(broadcastTables);
```

日志如下：

```
Actual SQL: ds0 ::: select * from t_config
```

##### 3.属于DMLStatement

属于`DMLStatement`同时分片条件为`AlwaysFalseShardingCondition`，或者没有指定表名，或者没有配置表规则；

```sql
select user_id,order_id from t_order where order_id = 101 and order_id = 102
```

where后面指定的条件会导致分片条件为`AlwaysFalseShardingCondition`

```sql
Actual SQL: ds1 ::: select user_id,order_id from t_order0 where order_id = 101 and order_id = 102
```

#### ShardingDataSourceGroupBroadcastRoutingEngine

数据源组广播，从数据源组中随机选择一个数据源；

##### 1.属于DALStatement

```sql
SHOW STATUS
```

`DALStatement`子类中除了use、set、reset、show databases；基本都会走此引擎；

```
Actual SQL: ds1 ::: SHOW STATUS
```

#### ShardingMasterInstanceBroadcastRoutingEngine

全实例路由用于 DCL 操作，授权语句针对的是数据库的实例。无论一个实例中包含多少个 Schema，每个数据库的实例只执行一次；

##### 1.属于DCLStatement

```sql
CREATE USER customer@127.0.0.1 identified BY '123'
```

**注**：这里的主实例会检测各实例之间，不能有相同的`hostname`和相同的`port`，本地测试同一台Mysql不同库，配置hostname不一致即可，比如localhost和127.0.0.1；

#### ShardingStandardRoutingEngine

标准路由是最常见的分片方式了，经过以上几种路由引擎的过滤，剩下的`SQLStatement`，就会走剩下的两个引擎了，我们配置分库分表策略，常规使用的增删改查都会使用此引擎；

##### 1.单表查询

```sql
select user_id,order_id from t_order where order_id = 101
```

以上使用常规配置，两个数据源分别是ds0,ds1；user_id作为分库键，order_id作为分表键；

```sql
Actual SQL: ds0 ::: select user_id,order_id from t_order1 where order_id = 101
Actual SQL: ds1 ::: select user_id,order_id from t_order1 where order_id = 101
```

101经过分片算法定位到物理表`t_order1`，但是无法定位数据库，所以分别到两个库执行；

##### 2.关联查询

```sql
select a.user_id,a.order_id from t_order a left join t_order_item b ON a.order_id=b.order_id where a.order_id = 101
```

如果是关联查询，则只有在两者配置了绑定关系，才会使用标准路由；

```java
Collection<String> bindingTables = new LinkedList<>();
bindingTables.add("t_order,t_order_item");
shardingRuleConfig.setBindingTableGroups(bindingTables);
```

以上配置了两张表为绑定表，关联查询与单表查询复杂度和性能相当，不会进行笛卡尔路由；

```sql
Actual SQL: ds0 ::: select a.user_id,a.order_id from t_order1 a left join t_order1 b ON a.order_id=b.order_id where a.order_id = 101
Actual SQL: ds1 ::: select a.user_id,a.order_id from t_order1 a left join t_order1 b ON a.order_id=b.order_id where a.order_id = 101
```

#### ShardingComplexRoutingEngine

笛卡尔路由是最复杂的情况，它无法根据绑定表的关系定位分片规则，因此非绑定表之间的关联查询需要拆解为笛卡尔积组合执行；

##### 1.关联查询

以上SQL如果不配置绑定关系，那么会进行笛卡尔路由，路由日志如下：

```sql
Actual SQL: ds0 ::: select a.user_id,a.order_id from t_order1 a left join t_order_item0 b ON a.order_id=b.order_id where a.order_id = 101
Actual SQL: ds0 ::: select a.user_id,a.order_id from t_order1 a left join t_order_item1 b ON a.order_id=b.order_id where a.order_id = 101
Actual SQL: ds1 ::: select a.user_id,a.order_id from t_order1 a left join t_order_item0 b ON a.order_id=b.order_id where a.order_id = 101
Actual SQL: ds1 ::: select a.user_id,a.order_id from t_order1 a left join t_order_item1 b ON a.order_id=b.order_id where a.order_id = 101
```



### 执行路由处理

经过以上流程处理，已经获取到了处理此`SQLStatement`对应的路由引擎，接下来只需要执行对应的路由引擎，获取路由结果即可；

```java
public interface ShardingRouteEngine {
    RouteResult route(ShardingRule shardingRule);
}
```

传入分片规则`ShardingRule`，返回路由结果`RouteResult`；下面以标准路由为例，来分析是如何执行路由处理的；`ShardingStandardRoutingEngine`的核心方法`getDataNodes`如下所示：

```java
    private Collection<DataNode> getDataNodes(final ShardingRule shardingRule, final TableRule tableRule) {
        if (isRoutingByHint(shardingRule, tableRule)) {
            return routeByHint(shardingRule, tableRule);
        }
        if (isRoutingByShardingConditions(shardingRule, tableRule)) {
            return routeByShardingConditions(shardingRule, tableRule);
        }
        return routeByMixedConditions(shardingRule, tableRule);
    }
```

以上有三种路由方式：hint方式路由、分片条件路由、混合条件路由；下面分别介绍；

#### Hint方式路由

首先判断是否使用强制路由方式：

```java
    private boolean isRoutingByHint(final ShardingRule shardingRule, final TableRule tableRule) {
        return shardingRule.getDatabaseShardingStrategy(tableRule) instanceof HintShardingStrategy && shardingRule.getTableShardingStrategy(tableRule) instanceof HintShardingStrategy;
    }
```

需要库和表路由策略都是`HintShardingStrategy`，这个只需要在配置`TableRuleConfiguration`分别配置数据库和表的策略都为`HintShardingStrategyConfiguration`即可；

```java
 private Collection<DataNode> route0(final ShardingRule shardingRule, final TableRule tableRule, final List<RouteValue> databaseShardingValues, final List<RouteValue> tableShardingValues) {
        Collection<String> routedDataSources = routeDataSources(shardingRule, tableRule, databaseShardingValues);
        Collection<DataNode> result = new LinkedList<>();
        for (String each : routedDataSources) {
            result.addAll(routeTables(shardingRule, tableRule, each, tableShardingValues));
        }
        return result;
    }
```

接下来就是分别执行路由库和路由表，不管是路由库还是表都需要两个核心的参数：当前可用的目标库或表、当前的分片值；

- 当前可用的目标库或表：这个就是初始化的库和表，比如目标库包括ds0、ds1，目标表包括t_order0、t_order1；
- 当前的分片值：当前的hint方式就是通过`HintManager`配置的分片值；

其他两种方式其实路由方式也都类似，只是分片值获取的方式不一样；有了这两个值就会调用我们自己定义的分库分表算法`ShardingAlgorithm`，这样就返回了经过路由后的库表，将结果保存到`DataNode`中：

```java
public final class DataNode {
    private final String dataSourceName;
    private final String tableName;
}
```

一个真实的库对应一个真实的表；最后将`DataNode`封装到`RouteResult`即可；

#### 分片条件路由

同样首先判断是否走分片条件路由：

```java
    private boolean isRoutingByShardingConditions(final ShardingRule shardingRule, final TableRule tableRule) {
        return !(shardingRule.getDatabaseShardingStrategy(tableRule) instanceof HintShardingStrategy || shardingRule.getTableShardingStrategy(tableRule) instanceof HintShardingStrategy);
    }
```

库和表路由策略都不是`HintShardingStrategy`的情况下才会走分片条件路由；

```java
    private Collection<DataNode> routeByShardingConditions(final ShardingRule shardingRule, final TableRule tableRule) {
        return shardingConditions.getConditions().isEmpty()
                ? route0(shardingRule, tableRule, Collections.emptyList(), Collections.emptyList()) : routeByShardingConditionsWithCondition(shardingRule, tableRule);
    }
```

然后会判断是否有`ShardingConditions`，关于`ShardingConditions`上面章节会专门介绍；如果没有`ShardingConditions`说明没有条件就会走全库表路由，如果有的话会从`ShardingConditions`中取出库表分片值，下面的逻辑就和Hint方式一样了；

#### 混合条件路由

混合模式就是库和表路由策略都不全都是`HintShardingStrategy`，要么表使用强制路由，要么库使用强制路由；

```java
    private Collection<DataNode> routeByMixedConditions(final ShardingRule shardingRule, final TableRule tableRule) {
        return shardingConditions.getConditions().isEmpty() ? routeByMixedConditionsWithHint(shardingRule, tableRule) : routeByMixedConditionsWithCondition(shardingRule, tableRule);
    }
```

会判断是否有`ShardingConditions`，如果没有说明库或者表路由有一个使用`HintShardingStrategy`，另外一个没有；否则就是Hint和Condition混合；这种情况就要看谁的优先级高了，很明显是Hint方式优先级高，可用看库分片值获取：

```java
   private List<RouteValue> getDatabaseShardingValues(final ShardingRule shardingRule, final TableRule tableRule, final ShardingCondition shardingCondition) {
        ShardingStrategy dataBaseShardingStrategy = shardingRule.getDatabaseShardingStrategy(tableRule);
        return isGettingShardingValuesFromHint(dataBaseShardingStrategy)
                ? getDatabaseShardingValuesFromHint() : getShardingValuesFromShardingConditions(shardingRule, dataBaseShardingStrategy.getShardingColumns(), shardingCondition);
    }
```

如果能获取到Hint分片值，那就使用Hint值，否则就从`Condition`中获取；



## 读写分离路由装饰器

经过上面分片路由装饰器的处理，根据优先级，如果配置了读写分离会执行读写分离装饰器`MasterSlaveRouteDecorator`；大致流程如下所示：

#### 读写分离配置

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

首先必须配置了读写分离策略ds0备库为ds01，ds1备库为ds11；

#### 库名称匹配

经过分片路由生成的`RouteUnit`中对应的库名称，和`MasterSlaveRule`中配置的名称能匹配；

#### 读写路由

```java
    public String route(final SQLStatement sqlStatement) {
        if (isMasterRoute(sqlStatement)) {
            MasterVisitedManager.setMasterVisited();
            return masterSlaveRule.getMasterDataSourceName();
        }
        return masterSlaveRule.getLoadBalanceAlgorithm().getDataSource(
                masterSlaveRule.getName(), masterSlaveRule.getMasterDataSourceName(), new ArrayList<>(masterSlaveRule.getSlaveDataSourceNames()));
    }
```

并不是配置了读写分离都会进行路由处理，有些`SQLStatement`是必须走主表的：

```java
private boolean isMasterRoute(final SQLStatement sqlStatement) {
        return containsLockSegment(sqlStatement) || !(sqlStatement instanceof SelectStatement) || MasterVisitedManager.isMasterVisited() || HintManager.isMasterRouteOnly();
    }
    
    private boolean containsLockSegment(final SQLStatement sqlStatement) {
        return sqlStatement instanceof SelectStatement && ((SelectStatement) sqlStatement).getLock().isPresent();
    }
```

- `SelectStatement`中包含了锁
- 非`SelectStatement`
- 配置了`MasterVisitedManager`，内部使用ThreadLocal管理
- 配置了`HintManager`，内部使用ThreadLocal管理

以上四种情况都是走主表，其他情况走备库，如果有多台备库，会进行负载均衡处理；

#### 替换路由库

经过读写分离路由处理之后，获取到库名称需要替换原理的库名称；

## 总结

本文主要介绍了`ShardingSphere-JDBC`的分片路由引擎，重点和难点在于不同类型的`SQLStatement`使用不同的路由引擎，路由引擎众多，本文重点举例介绍了每种路由引擎在何种条件下使用；经过路由引擎会获取到路由结果下面就是对SQL进行改写了，改写成真实的库表，这样数据库才能执行。

## 参考

https://shardingsphere.apache.org/document/current/cn/overview/

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。
