## 前言

上文[ShardingSphere-JDBC分片路由引擎](https://juejin.cn/post/6956182616181571620)中介绍了分片流程中的路由引擎，最终获取了路由结果；本文要介绍的改写引擎需要使用路由结果来对SQL进行改写，改写成可以被正确的分库分表能够执行的SQL；这里面涉及对各种SQL改写的情况众多，接下来本文会进行一一分析。

## 改写装饰器

重写引擎同样使用了装饰器模式，提供了接口类`SQLRewriteContextDecorator`，实现类包括：

- ShardingSQLRewriteContextDecorator：分片SQL改写装饰器；
- ShadowSQLRewriteContextDecorator：影子库SQL改写装饰器；
- EncryptSQLRewriteContextDecorator：数据加密SQL改写装饰器；

默认加载`ShardingSQLRewriteContextDecorator`和`EncryptSQLRewriteContextDecorator`，使用`java.util.ServiceLoader`来加载重写装饰器，需要在`META-INF/services/`中指定具体的实现类：

```
org.apache.shardingsphere.sharding.rewrite.context.ShardingSQLRewriteContextDecorator
org.apache.shardingsphere.encrypt.rewrite.context.EncryptSQLRewriteContextDecorator
```

装饰器可以叠加，所以提供了优先级功能`OrderAware`，同时每个装饰器都有对应的规则，大致如下所示：

| 装饰器-SQLRewriteContextDecorator  | 规则-BaseRule | 优先级-Order |
| ---------------------------------- | ------------- | ------------ |
| ShardingSQLRewriteContextDecorator | ShardingRule  | 0            |
| EncryptSQLRewriteContextDecorator  | EncryptRule   | 20           |
| ShadowSQLRewriteContextDecorator   | ShadowRule    | 30           |

只有在配置了相关`BaseRule`，对应的`SQLRewriteContextDecorator`才能生效，最常见的是`ShardingSQLRewriteContextDecorator`，下面重点介绍此装饰器；

## 改写引擎

不同的SQL语句，需要处理的改写都不一样，改写引擎的整体结构划分如下图所示(来自官网)：

![rewrite_architecture_cn.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1d306bdebb54024bcc131ab86142e64~tplv-k3u1fbpfcp-watermark.image)

执行改写引擎之前需要做一些准备工作，整个改写流程大致分为以下几步：

- 根据不同的改写装饰器构造不同的`SQLTokenGenerator`列表；
- 根据`SQLTokenGenerator`生成对应的`SQLToken`；
- 改写引擎根据`SQLToken`执行改写操作；

### 构造SQLTokenGenerator

不同的装饰器需要构造不同的`SQLTokenGenerator`列表，以最常见的`ShardingSQLRewriteContextDecorator`为例，会准备如下13种`SQLTokenGenerator`：

```java
    private Collection<SQLTokenGenerator> buildSQLTokenGenerators() {
        Collection<SQLTokenGenerator> result = new LinkedList<>();
        addSQLTokenGenerator(result, new TableTokenGenerator());
        addSQLTokenGenerator(result, new DistinctProjectionPrefixTokenGenerator());
        addSQLTokenGenerator(result, new ProjectionsTokenGenerator());
        addSQLTokenGenerator(result, new OrderByTokenGenerator());
        addSQLTokenGenerator(result, new AggregationDistinctTokenGenerator());
        addSQLTokenGenerator(result, new IndexTokenGenerator());
        addSQLTokenGenerator(result, new OffsetTokenGenerator());
        addSQLTokenGenerator(result, new RowCountTokenGenerator());
        addSQLTokenGenerator(result, new GeneratedKeyInsertColumnTokenGenerator());
        addSQLTokenGenerator(result, new GeneratedKeyForUseDefaultInsertColumnsTokenGenerator());
        addSQLTokenGenerator(result, new GeneratedKeyAssignmentTokenGenerator());
        addSQLTokenGenerator(result, new ShardingInsertValuesTokenGenerator());
        addSQLTokenGenerator(result, new GeneratedKeyInsertValuesTokenGenerator());
        return result;
    }
```

以上实现类的公共接口类为`SQLTokenGenerator`，提供了是否生效的公共方法：

```java
public interface SQLTokenGenerator {
    boolean isGenerateSQLToken(SQLStatementContext sqlStatementContext);
}
```

各种不同的`SQLTokenGenerator`并不是每次都能生效的，需要根据不同SQL语句进行判断，SQL语句在解析引擎中已经被解析为`SQLStatementContext`，这样可以通过`SQLStatementContext`参数进行判断；

#### TableTokenGenerator

`TableToken`生成器，主要用于对表名的改写；

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return true;
    }
```

可以发现这里对是否生成`SQLToken`没有任何条件，直接返回true；但是在生成`TableToken`的时候是会检查是否存在表信息以及是否配置相关表`TableRule`；

#### DistinctProjectionPrefixTokenGenerator

`DistinctProjectionPrefixToken`生成器，主要对聚合函数和去重的处理：

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext && !((SelectStatementContext) sqlStatementContext).getProjectionsContext().getAggregationDistinctProjections().isEmpty();
    }
```

首先必须是`select`语句，同时包含：聚合函数和`Distinct`去重，比如下面的SQL：

```sql
select sum(distinct user_id) from t_order where order_id = 101
```

改写之后的SQL如下所示：

```sql
Actual SQL: ds0 ::: select DISTINCT user_id AS AGGREGATION_DISTINCT_DERIVED_0 from t_order1 where order_id = 101
Actual SQL: ds1 ::: select DISTINCT user_id AS AGGREGATION_DISTINCT_DERIVED_0 from t_order1 where order_id = 101
```

#### ProjectionsTokenGenerator

`ProjectionsToken`生成器，聚合函数需要做派生处理，比如`AVG`函数

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext && !getDerivedProjectionTexts((SelectStatementContext) sqlStatementContext).isEmpty();
    }
    
    private Collection<String> getDerivedProjectionTexts(final SelectStatementContext selectStatementContext) {
        Collection<String> result = new LinkedList<>();
        for (Projection each : selectStatementContext.getProjectionsContext().getProjections()) {
            if (each instanceof AggregationProjection && !((AggregationProjection) each).getDerivedAggregationProjections().isEmpty()) {
                result.addAll(((AggregationProjection) each).getDerivedAggregationProjections().stream().map(this::getDerivedProjectionText).collect(Collectors.toList()));
            } else if (each instanceof DerivedProjection) {
                result.add(getDerivedProjectionText(each));
            }
        }
        return result;
    }
```

首先必须是`select`语句，其次可以是：

- 聚合函数，同时需要派生新的函数，比如avg函数；
- 派生关键字，比如order by，group by等；

比如`avg`函数使用：

```sql
select avg(user_id) from t_order where order_id = 101
```

改写的SQL如下所示：

```sql
Actual SQL: ds0 ::: select avg(user_id) , COUNT(user_id) AS AVG_DERIVED_COUNT_0 , SUM(user_id) AS AVG_DERIVED_SUM_0 from t_order1 where order_id = 101
Actual SQL: ds1 ::: select avg(user_id) , COUNT(user_id) AS AVG_DERIVED_COUNT_0 , SUM(user_id) AS AVG_DERIVED_SUM_0 from t_order1 where order_id = 101
```

比如`order by`使用：

```sql
select user_id from t_order order by order_id
```

改写的SQL如下所示：

```sql
Actual SQL: ds0 ::: select user_id , order_id AS ORDER_BY_DERIVED_0 from t_order0 order by order_id
Actual SQL: ds0 ::: select user_id , order_id AS ORDER_BY_DERIVED_0 from t_order1 order by order_id
Actual SQL: ds1 ::: select user_id , order_id AS ORDER_BY_DERIVED_0 from t_order0 order by order_id
Actual SQL: ds1 ::: select user_id , order_id AS ORDER_BY_DERIVED_0 from t_order1 order by order_id
```

需要`order by`中指定的排序字段在`select`后面没有，这时候会派生一个；

#### OrderByTokenGenerator

`OrderByToken`生成器，自动生成order by，生效条件：

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext && ((SelectStatementContext) sqlStatementContext).getOrderByContext().isGenerated();
    }
```

首先必须是`select`语句，其次有自动生成的`order by`内容；

比如如下SQL：

```sql
select distinct user_id from t_order
```

改写的SQL如下所示，改写自动添加了`ORDER BY`

```sql
Actual SQL: ds0 ::: select distinct user_id from t_order0 ORDER BY user_id ASC 
Actual SQL: ds0 ::: select distinct user_id from t_order1 ORDER BY user_id ASC 
Actual SQL: ds1 ::: select distinct user_id from t_order0 ORDER BY user_id ASC 
Actual SQL: ds1 ::: select distinct user_id from t_order1 ORDER BY user_id ASC 
```

#### AggregationDistinctTokenGenerator

`AggregationDistinctToken`生成器，类似`DistinctProjectionPrefixTokenGenerator`

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext;
    }
```

是否生成`SQLToken`没做检查，但是在生成`SQLToken`的时候会检查是否有聚合函数和`Distinct`去重；

#### IndexTokenGenerator

`IndexToken`生成器，主要用在使用索引的地方，对索引名称重命名，用在sql server，PostgreSQL中，Mysql并没有使用；

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof IndexAvailable && !((IndexAvailable) sqlStatementContext).getIndexes().isEmpty();
    }
```

#### OffsetTokenGenerator

`OffsetToken`生成器，主要作用于分页，对应`limit`的`offset`关键字

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext
                && ((SelectStatementContext) sqlStatementContext).getPaginationContext().getOffsetSegment().isPresent()
                && ((SelectStatementContext) sqlStatementContext).getPaginationContext().getOffsetSegment().get() instanceof NumberLiteralPaginationValueSegment;
    }
```

如下通过`limit`实现分页查询：

```sql
SELECT * FROM t_order LIMIT 1,2
```

改写的SQL如下所示，分页参数改写成了0，3

```sql
Actual SQL: ds0 ::: SELECT * FROM t_order0 LIMIT 0,3
Actual SQL: ds0 ::: SELECT * FROM t_order1 LIMIT 0,3
Actual SQL: ds1 ::: SELECT * FROM t_order0 LIMIT 0,3
Actual SQL: ds1 ::: SELECT * FROM t_order1 LIMIT 0,3
```

#### RowCountTokenGenerator

`RowCountToken`生成器，同样作用于分页，对应`limit`的`count`关键字

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof SelectStatementContext
                && ((SelectStatementContext) sqlStatementContext).getPaginationContext().getRowCountSegment().isPresent()
                && ((SelectStatementContext) sqlStatementContext).getPaginationContext().getRowCountSegment().get() instanceof NumberLiteralPaginationValueSegment;
    }
```

实例和上面的`OffsetTokenGenerator`一致；

#### GeneratedKeyForUseDefaultInsertColumnsTokenGenerator

`UseDefaultInsertColumnsToken`生成器，insert的sql中并未包含表的列名称

```java
    public final boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof InsertStatementContext && ((InsertStatementContext) sqlStatementContext).getGeneratedKeyContext().isPresent()
                && ((InsertStatementContext) sqlStatementContext).getGeneratedKeyContext().get().isGenerated() && isGenerateSQLToken(((InsertStatementContext) sqlStatementContext).getSqlStatement());
    }
    
    protected boolean isGenerateSQLToken(final InsertStatement insertStatement) {
        return insertStatement.useDefaultColumns();
    }
```

使用以上TokenGenerator需要多个条件包括：

- 必须是insert语句；

- 配置了`KeyGeneratorConfiguration`，如下所示：

  ```java
  orderTableRuleConfig.setKeyGeneratorConfig(new KeyGeneratorConfiguration("SNOWFLAKE", "id"));
  ```

- 主键是自动生成的也就是用户没有主动生成主键；

- 使用默认的字段，insert的sql中并未包含表的列名称；

下面看一条insert sql的实例：

```sql
insert into t_order values (1,1)
```

改写的SQL如下所示：

```sql
Actual SQL: ds1 ::: insert into t_order1(user_id, order_id, id) values (1, 1, 600986707608731648)
```

#### GeneratedKeyInsertColumnTokenGenerator

`GeneratedKeyInsertColumnToken`生成器，insert的sql中包含表的列名称，其他基本同上

```java
    protected boolean isGenerateSQLToken(final InsertStatement insertStatement) {
        Optional<InsertColumnsSegment> sqlSegment = insertStatement.getInsertColumns();
        return sqlSegment.isPresent() && !sqlSegment.get().getColumns().isEmpty();
    }
```

下面看一条insert sql的实例：

```sql
insert into t_order (user_id,order_id) values (1,1)
```

改写的SQL如下所示：

```sql
Actual SQL: ds1 ::: insert into t_order1 (user_id,order_id, id) values (1, 1, 600988204400640000)
```

#### GeneratedKeyAssignmentTokenGenerator

`GeneratedKeyAssignmentToken`生成器，主要用于`insert...set`操作，同上无需指定主键

```java
    protected boolean isGenerateSQLToken(final InsertStatement insertStatement) {
        return insertStatement.getSetAssignment().isPresent();
    }
```

下面看一条insert set的实例：

```sql
insert into t_order set user_id = 111,order_id=111
```

改写的SQL如下所示：

```sql
Actual SQL: ds1 ::: insert into t_order1 set user_id = 111,order_id=111, id = 600999588391813120
```

#### ShardingInsertValuesTokenGenerator

`InsertValuesToken`生成器，有插入值做分片处理

```java
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof InsertStatementContext && !(((InsertStatementContext) sqlStatementContext).getSqlStatement()).getValues().isEmpty();
    }
```

下面看一条insert sql实例：

```sql
insert into t_order values (95,1,1),(96,2,2)
```

改写的SQL如下所示：

```sql
Actual SQL: ds1 ::: insert into t_order1 values (95, 1, 1)
Actual SQL: ds0 ::: insert into t_order0 values (96, 2, 2)
```

#### GeneratedKeyInsertValuesTokenGenerator

`InsertValuesToken`生成器，只要有插入值，就会存在此生成器，前提不能指定主键，使用自动生成

```java
    protected boolean isGenerateSQLToken(final InsertStatement insertStatement) {
        return !insertStatement.getValues().isEmpty();
    }
```

下面看一条insert sql实例：

```sql
insert into t_order values (1,1),(2,2)
```

改写的SQL如下所示：

```sql
Actual SQL: ds1 ::: insert into t_order1(user_id, order_id, id) values (1, 1, 601005570564030465)
Actual SQL: ds0 ::: insert into t_order0(user_id, order_id, id) values (2, 2, 601005570564030464)
```

### 生成SQLToken

以上分别介绍了常见的几种`TokenGenerator`，以及在哪种条件下可以生成`SQLToken`，本节重点看一下是如何生成`SQLToken`的，分别提供了两个接口类：

- CollectionSQLTokenGenerator：对应的`TokenGenerator`会生成`SQLToken`列表；实现类包括：`AggregationDistinctTokenGenerator`、`TableTokenGenerator`、`IndexTokenGenerator`等
- OptionalSQLTokenGenerator：对应的`TokenGenerator`会生成唯一的`SQLToken`；实现类包括：`DistinctProjectionPrefixTokenGenerator`、`ProjectionsTokenGenerator`、`OrderByTokenGenerator`、`OffsetTokenGenerator`、`RowCountTokenGenerator`、`GeneratedKeyForUseDefaultInsertColumnsTokenGenerator`、`GeneratedKeyInsertColumnTokenGenerator`、`GeneratedKeyAssignmentTokenGenerator`、`ShardingInsertValuesTokenGenerator`、`GeneratedKeyInsertValuesTokenGenerator`等；

```java
public interface CollectionSQLTokenGenerator<T extends SQLStatementContext> extends SQLTokenGenerator {
    Collection<? extends SQLToken> generateSQLTokens(T sqlStatementContext);
}

public interface OptionalSQLTokenGenerator<T extends SQLStatementContext> extends SQLTokenGenerator {
    SQLToken generateSQLToken(T sqlStatementContext);
}
```

下面重点分析一下每种生成器是如何生成对应的`SQLToken`的；

#### TableTokenGenerator

首先从解析引擎中获取的`SQLStatementContext`中获取所有表信息`SimpleTableSegment`，然后查看当前表是否配置了`TableRule`，如果都能满足那么会创建一个`TableToken`：

```java
public final class TableToken extends SQLToken implements Substitutable, RouteUnitAware {
    @Getter
    private final int stopIndex;//结束位置，开始位置在父类SQLToken中
    private final IdentifierValue identifier; //表标识符信息 
    private final SQLStatementContext sqlStatementContext;//SQLStatement上下文
    private final ShardingRule shardingRule;//定义的分片规则
}
```

如果SQL中存在多个表信息，这里会生成`TableToken`列表；因为需要做表名称替换处理，所以需要如上定义的相关参数，方便后续做重写处理；

#### DistinctProjectionPrefixTokenGenerator

去重处理改写后的SQL需要在指定位置添加`DISTINCT`关键字：

```java
public final class DistinctProjectionPrefixToken extends SQLToken implements Attachable {
    public DistinctProjectionPrefixToken(final int startIndex) {
        super(startIndex);
    }
}
```

这里只需要提供一个开始位置即可，也就是开始插入`DISTINCT`关键字的位置；

#### ProjectionsTokenGenerator

聚合函数和派生关键字生成器，可以查看枚举类`DerivedColumn`：

```java
public enum DerivedColumn {
    AVG_COUNT_ALIAS("AVG_DERIVED_COUNT_"), 
    AVG_SUM_ALIAS("AVG_DERIVED_SUM_"), 
    ORDER_BY_ALIAS("ORDER_BY_DERIVED_"), 
    GROUP_BY_ALIAS("GROUP_BY_DERIVED_"),
    AGGREGATION_DISTINCT_DERIVED("AGGREGATION_DISTINCT_DERIVED_");
}
```

比如上面介绍的实例中`avg`会派生成：

- COUNT(user_id) AS AVG_DERIVED_COUNT_0 ；
- SUM(user_id) AS AVG_DERIVED_SUM_0 ；

最后所有派生保存到一个文本中，封装到`ProjectionsToken`中：

```java
public final class ProjectionsToken extends SQLToken implements Attachable {
    private final Collection<String> projections;//派生列表
    public ProjectionsToken(final int startIndex, final Collection<String> projections) {
        super(startIndex);
        this.projections = projections;
    }
}
```

#### OrderByTokenGenerator

重点是生成`OrderByContext`类，具体可以查看`OrderByContextEngine`，其中会判断`isDistinctRow`；最终会把所有需要生成`order by`的字段和排序方式保存到`OrderByToken`中：

```java
public final class OrderByToken extends SQLToken implements Attachable {
    private final List<String> columnLabels = new LinkedList<>();  //order by字段
    private final List<OrderDirection> orderDirections = new LinkedList<>();//排序方式
    public OrderByToken(final int startIndex) {
        super(startIndex);
    }
}
```

#### AggregationDistinctTokenGenerator

条件和`DistinctProjectionPrefixTokenGenerator`一致，前者主要通过`DistinctProjectionPrefixToken`生成`DISTINCT`关键字；而此生成器是给相关字段添加派生的别名，如上面实例中的`AGGREGATION_DISTINCT_DERIVED_0`

```sql
select DISTINCT user_id AS AGGREGATION_DISTINCT_DERIVED_0 from t_order1 where order_id = 101
```

```java
public final class AggregationDistinctToken extends SQLToken implements Substitutable {
    private final String columnName;//字段名称
    private final String derivedAlias;//别名
}
```

#### IndexTokenGenerator

如果SQL是一个`IndexAvailable`，并且包含索引信息，则会生成一个`IndexToken`，其中信息和`TableToken`一致；常见的`IndexAvailable`包含：`AlterIndexStatementContext`、`CreateIndexStatementContext`、`CreateTableStatementContext`、`DropIndexStatementContext`；

#### OffsetTokenGenerator

主要是对`limit`关键字中的`offset`值进行重置处理，处理的信息包含在`OffsetToken`中：

```java
public final class OffsetToken extends SQLToken implements Substitutable {
    @Getter
    private final int stopIndex;
    private final long revisedOffset; //修订过的offset
}
```

#### RowCountTokenGenerator

主要是对`limit`关键字中的`count`值进行重置处理，处理的信息包含在`RowCountToken`中：

```java
public final class RowCountToken extends SQLToken implements Substitutable {
    @Getter
    private final int stopIndex;
    private final long revisedRowCount; //修订过的rowcout
}
```

#### GeneratedKeyForUseDefaultInsertColumnsTokenGenerator

因为配置了`KeyGeneratorConfiguration`，insert语句中会自动生成组件，解析的时候会在`InsertStatementContext`中生成`GeneratedKeyContext`，里面包含了主键字段，以及主键值；此生成器是在insert中没有指定字段的情况，所有字段会被保存到`UseDefaultInsertColumnsToken`中，字段列表是有序的，需要将生成的id移动到列表的末尾；

```java
public final class UseDefaultInsertColumnsToken extends SQLToken implements Attachable {
    private final List<String> columns;//字段列表：user_id,order_id,id
}
```

#### GeneratedKeyInsertColumnTokenGenerator

此生成器是在insert中指定了字段的情况，需要改写的地方是添加主键字段名称即可，保存到`GeneratedKeyInsertColumnToken`中：

```java
public final class GeneratedKeyInsertColumnToken extends SQLToken implements Attachable {
    private final String column;//主键字段：id
}
```

#### GeneratedKeyAssignmentTokenGenerator

此生成器是在`insert  set`中使用，需要添加主键、值，但是因为可以由用户指定parameter，所以这里会根据是否配置了parameter来生成不同的Token：

- LiteralGeneratedKeyAssignmentToken：没有parameter的情况，提供主键名称和主键值：

  ```java
  public final class LiteralGeneratedKeyAssignmentToken extends GeneratedKeyAssignmentToken {
      private final Object value;//主键值
      public LiteralGeneratedKeyAssignmentToken(final int startIndex, final String columnName, final Object value) {
          super(startIndex, columnName);//开始位置和主键名称
          this.value = value;
      }
  ```

- ParameterMarkerGeneratedKeyAssignmentToken：指定了parameter，只需要提供主键名称即可，值从parameter中获取：

  ```java
  public final class ParameterMarkerGeneratedKeyAssignmentToken extends GeneratedKeyAssignmentToken {
      public ParameterMarkerGeneratedKeyAssignmentToken(final int startIndex, final String columnName) {
          super(startIndex, columnName);//开始位置和主键名称
      }
  }
  ```

#### ShardingInsertValuesTokenGenerator

插入的值可以是一条也可以是多条，每条数据都会和`DataNode`绑定，也就是属于哪个库属于哪个表，这里数据和`DataNode`的绑定被包装到了`ShardingInsertValue`中：

```java
public final class ShardingInsertValue {
    private final Collection<DataNode> dataNodes;//数据节点信息
    private final List<ExpressionSegment> values;//数据信息
}
```

最后所有数据包装到`ShardingInsertValuesToken`中；

#### GeneratedKeyInsertValuesTokenGenerator

此生成器会对前面`ShardingInsertValuesTokenGenerator`生成的`ShardingInsertValue`进行再加工处理，主要针对没有指定主键的情况，对其中的`values`增加一个`ExpressionSegment`保存主键信息；

### 执行改写

通过以上两步已经准备好了所有`SQLToken`，下面就可以执行改写操作了，提供了改写引擎`SQLRouteRewriteEngine`，两个重要的参数分别是：

- SQLRewriteContext：SQL改写上下文，生成的SQLToken都存在上下文中；
- RouteResult：路由引擎产生的结果；

有了以上两个核心参数就可以执行改写操作了：

```java
    public Map<RouteUnit, SQLRewriteResult> rewrite(final SQLRewriteContext sqlRewriteContext, final RouteResult routeResult) {
        Map<RouteUnit, SQLRewriteResult> result = new LinkedHashMap<>(routeResult.getRouteUnits().size(), 1);
        for (RouteUnit each : routeResult.getRouteUnits()) {
            result.put(each, new SQLRewriteResult(new RouteSQLBuilder(sqlRewriteContext, each).toSQL(), getParameters(sqlRewriteContext.getParameterBuilder(), routeResult, each)));
        }
        return result;
    }
```

遍历每个路由单元`RouteUnit`，每个路由单元都对应一条SQL语句；根据路由单元和`SQLToken`列表生成改写这条SQL；可以发现这里执行了`RouteSQLBuilder`中的toSQL方法：

```java
    public final String toSQL() {
        if (context.getSqlTokens().isEmpty()) {
            return context.getSql();
        }
        Collections.sort(context.getSqlTokens());
        StringBuilder result = new StringBuilder();
        result.append(context.getSql().substring(0, context.getSqlTokens().get(0).getStartIndex()));
        for (SQLToken each : context.getSqlTokens()) {
            result.append(getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
        return result.toString();
    }
    
    protected String getSQLTokenText(final SQLToken sqlToken) {
        if (sqlToken instanceof RouteUnitAware) {
            return ((RouteUnitAware) sqlToken).toString(routeUnit);
        }
        return sqlToken.toString();
    }
	
    private String getConjunctionText(final SQLToken sqlToken) {
        return context.getSql().substring(getStartIndex(sqlToken), getStopIndex(sqlToken));
    }
```

首先对`SQLToken`列表进行排序，排序根据`startIndex`从小到大排序；然后从截取从0位置到第一个`SQLToken`的开始位置，这一部分的sql是无需任何改动的；接下来就是遍历`SQLToken`，会判断`SQLToken`是否为`RouteUnitAware`，如果是会做路由替换，比如物理表替换逻辑表；最后在截取两个`SQLToken`的中间部分，遍历所有`SQLToken`，返回重新拼接的SQL；

以一条查询SQL为例：

```sql
select user_id,order_id from t_order where order_id = 101
```

第一次截取从0位置到第一个`SQLToken`的开始位置，结果为：

```sql
select user_id,order_id from 
```

然后遍历`SQLToken`，当前只有一个`TableToken`，并且是一个`RouteUnitAware`，表格会被替换处理：

```sql
t_order->t_order1
```

截取最后剩余的部分：

```sql
 where order_id = 101
```

然后把每个部分拼接到一起，组成能被数据库执行的SQL：

```sql
select user_id,order_id from t_order1 where order_id = 101
```

下面简单介绍一下每种`SQLToken`是如何执行改写操作的；

#### TableToken

`TableToken`是一个`RouteUnitAware`，所以在重写的时候传入了路由单元`RouteUnit`，需要根据路由单元来决定逻辑表应该改写为哪个物理表；

#### DistinctProjectionPrefixToken

主要对聚合函数和去重的处理，这里只是把当前`Token`，改写成一个`DISTINCT` ，这里其实只是做了部分处理，聚合函数并没有体现，所以整个流程中还有归并这流程，很多聚合函数都需要在归并流程中处理；

#### ProjectionsToken

聚合函数需要做派生处理，只需要把派生的字符，拼接起来即可，比如：AVG函数派生的`AVG_DERIVED_COUNT`、`AVG_DERIVED_SUM`，同样AVG聚合函数同样需要到归并流程中处理；

#### OrderByToken

对里面保存的`order by`字段和对应的排序方式，进行遍历处理，结果类似如下：

```sql
order by column1 desc,column2 desc...
```

#### AggregationDistinctToken

主要对聚合函数和去重的处理，`DistinctProjectionPrefixToken`拼接`DISTINCT` 关键字，此Token拼接别名；组合起来如下所示：

```sql
select DISTINCT user_id AS AGGREGATION_DISTINCT_DERIVED_0 
```

#### IndexToken

`IndexToken`是一个`RouteUnitAware`，根据路由单元改写索引名称；

#### OffsetToken

通过修订过的offset，改写limit中的offset；

#### RowCountToken

通过修订过的RowCount，改写limit中的count；

#### GeneratedKeyInsertColumnToken

这个改写只是针对字段名称，并不是值，值是需要做路由处理的，即在现有的字段列表中添加主键名称；

```
(user_id)->(user_id,id)
```

#### UseDefaultInsertColumnsToken

此种模式是在插入数据的时候没有指定任何字段，所有这里会生成所有字段名称，同时会添加主键名称

`()->(user_id,id)`

#### GeneratedKeyAssignmentToken

上小节中介绍了此Token包含两个子类，分别对应insert set有参数和无参数的情况；这里的改写是包含名称和值的

```
set user_id = 11,id = 1234567
```

#### InsertValuesToken

这里区别以上`GeneratedKeyInsertColumnToken`和`UseDefaultInsertColumnsToken`对字段的处理，此Token是对值的处理，值肯定要路由处理，所有此Token也是`RouteUnitAware`；

## 总结

本文重点介绍了需要改写SQL的13种情况，其实就是找出所有需要被改写的SQL，然后记录到不同的Token中，同时还会记录当前Token在原SQL中对应的startIndex和stopIndex，这样可以做改写处理，替换到原理位置的SQL；通过遍历整个Token列表最终达到改写SQL的所有部位；当然整个改写有些地方是改变了原来SQL的含义比如聚合函数，所以ShardingSphere-JDBC还提供了专门的归并引擎，用来保证SQL的完整性。

## 参考

https://shardingsphere.apache.org/document/current/cn/overview/

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。
