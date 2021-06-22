## 前言

上文[ShardingSphere-JDBC入门实战](https://juejin.cn/post/6953054143321735198)中对ShardingSphere-JDBC如何使用做了简单介绍，接下来打算从源码层面对数据分片做更加详细的介绍，整个数据分片会经过一个复杂的流程包括：解析、路由、改写、执行、归并这几个子流程，每个子流程都有对应的引擎来处理，本文重点分析子流程中的解析引擎。


## 分片流程

在介绍解析引擎之前，我们对各个子流程做一个简单的介绍；我们可以想象一下大概要经过几个流程；首先用户操作的都是逻辑表，最终是要被替换成物理表的，所以需要对SQL进行解析，其实就是理解SQL；然后就是根据分片路由算法，应该路由到哪个表哪个库；接下来需要生成真实的SQL，这样SQL才能被执行；生成的SQL可能有多条，每条都要执行；最后把多条执行的结果进行归并，返回结果集；整个流程大致如下(来自官网)：


![sharding_architecture_cn.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad3443c20d9b4426abb62041066fc7ab~tplv-k3u1fbpfcp-watermark.image)

由 `SQL 解析 => 执行器优化 => SQL 路由 => SQL 改写 => SQL 执行 => 结果归并`的流程组成；每个子流程都有专门的引擎：

- SQL解析：分为词法解析和语法解析。 先通过词法解析器将 SQL 拆分为一个个不可再分的单词。再使用语法解析器对 SQL 进行理解，并最终提炼出解析上下文。 解析上下文包括表、选择项、排序项、分组项、聚合函数、分页信息、查询条件以及可能需要修改的占位符的标记；
- 执行器优化：合并和优化分片条件，如 OR 等；
- SQL路由：根据解析上下文匹配用户配置的分片策略，并生成路由路径；目前支持分片路由和广播路由；
- SQL改写：将 SQL 改写为在真实数据库中可以正确执行的语句。SQL 改写分为正确性改写和优化改写；
- SQL执行：通过多线程执行器异步执行；
- 结果归并：将多个执行结果集归并以便于通过统一的 JDBC 接口输出。结果归并包括流式归并、内存归并和使用装饰者模式的追加归并这几种方式。

本文重点分析SQL解析部分，但是在分析之前我们需要大致了解其中的ANTLR核心组件；



## 关于ANTLR

ANTLR (Another Tool for Language Recognition) 是一个强大的解析器的生成器，可以用来读取、处理、执行或翻译结构化文本或二进制文件。他被广泛用来构建语言，工具和框架。ANTLR可以从语法上来生成一个可以构建和遍历解析树的解析器。

ANTLR官方地址：https://www.antlr.org

ANTLR由两部分组成：

- 将用户自定义语法翻译成Java中的解析器/词法分析器的工具，对应antlr-complete.jar；
- 解析器运行时需要的环境库文件，对应antlr-runtime.jar；

### ANTLR语法

ANTLR默认是一个已.g4结尾的文件，一个语法定义文件一般来说有一个通用的结构如下：

```
/** Optional javadoc style comment */ 
grammar Name; ① 
options {...} 
import ... ; 

tokens {...} 
channels {...} // lexer only 
@actionName {...} 

rule1 // parser and lexer rules, possibly intermingled 
... 
ruleN
```

- grammar：语法名称，必须和文件名一致；可以包含前缀lexer和parser，如下所示：

  ```
  lexer grammar MySqlLexer;
  parser grammar MySqlParser;
  ```

- options：可以在语法和规则元素级别指定许多选项，grammar可以包含：superClass、language、tokenVocab、TokenLabelType、contextSuperClass等，比如

  ```
  options { tokenVocab=MySqlLexer; }
  ```

- import：将一个语法分割成多个逻辑上的、可复用的块，有点类似超类；

- tokens：为那些没有关联词法规则的`grammar`来定义`tokens`的类型；

  ```xml
  // explicitly define keyword token types to avoid implicit definition warnings
  tokens { BEGIN, END, IF, THEN, WHILE }
   
  @lexer::members { // keywords map used in lexer to assign token types
  Map<String,Integer> keywords = new HashMap<String,Integer>() {{
  	put("begin", KeywordsParser.BEGIN);
  	put("end", KeywordsParser.END);
  	...
  }};
  }
  ```

- channels：只有lexer(词法分析)的`grammar`才能包含自定义的`channels`，比如：

  ```
  channels {
    WHITESPACE_CHANNEL,
    COMMENTS_CHANNEL
  }
  ```

  以上`channels`可以在lexer(词法分析)规则中像枚举一样使用：

  ```
  WS : [ \r\t\n]+ -> channel(WHITESPACE_CHANNEL) ;
  ```

- actionName：目前只有两个定义的命名操作（针对Java目标）在语法规则之外使用：`header`和`members`；前者在识别器类定义之前将代码注入到生成的识别器类文件中，后者将代码作为字段和方法注入到识别器类定义中。

- rule：规则可以分为：Lexer Rules和Parser Rules；规则格式如下所示：

  ```
  ​```
  ruleName : alternative1 | ... | alternativeN ;
  ​```
  ```

  Lexer Rules：名称以大写字母开头；

  Parser Rules：名称以小写字母开头；

更多参考官方文档：[https://github.com/antlr/antlr4/blob/master/doc/index.md](web://urllink//https://github.com/antlr/antlr4/blob/master/doc/index.md)

### ANTLR使用

#### 配置环境

首先需要去官网下载antlr-complete.jar文件，我这里使用的版本是：4.7.2；然后需要配置`CLASSPATH`：

```
.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar;E:\antlr\antlr-4.7.2-complete.jar
```

检测一下是否成功：

```
E:\antlr>java org.antlr.v4.Tool
ANTLR Parser Generator  Version 4.7.2
 -o ___              specify output directory where all output is generated
 -lib ___            specify location of grammars, tokens files
 -atn                generate rule augmented transition network diagrams
 ......
```

#### 语法文件

我们需要根据ANTLR提供的语法定义自己的语法文件，比如Hello.g4如下所示：

```
// Define a grammar called Hello
grammar Hello;
r  : 'hello' ID ;         // match keyword hello followed by an identifier
ID : [a-z]+ ;             // match lower-case identifiers
WS : [ \t\r\n]+ -> skip ; // skip spaces, tabs, newlines
```

#### 处理语法文件

使用ANTLR执行如下命令：

```
E:\antlr>java -jar antlr-4.7.2-complete.jar Hello.g4
```

会在当前目录下生成如下一堆文件：

```
HelloParser.java
HelloLexer.java
HelloListener.java
HelloBaseListener.java
HelloLexer.tokens
Hello.tokens
HelloLexer.interp
Hello.interp
```

#### 测试

首先需要编译上面生成的java类：

```
E:\antlr>javac Hello*.java
```

通过如下命令，展示树形图形：

```
E:\antlr>java org.antlr.v4.gui.TestRig Hello  r -gui
hello zhaohui
^Z
```

注：最后的结尾unix使用ctrl+D，windows使用ctrl+Z；


![image-20210422154104797.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cef1b91f823a489cb17d6bcaca9b1304~tplv-k3u1fbpfcp-watermark.image)

#### 插件方式

除了以上方式还可以直接在IDE中使用插件，各种IDE的插件地址可以直接在官网查看：

插件地址：https://www.antlr.org/tools.html

##### 处理语法文件

在Hello.g4文件上右击“Configure Antlr...”，如下所示：


![image-20210422163020303.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf0130da7d7e43aab22e79060a3bdd11~tplv-k3u1fbpfcp-watermark.image)

其中几个比较重要的配置包括：生成文件输出的位置、生成类指定的包名、语法树遍历的模式；

语法树遍历的模式其中可以配置两种模式：listener模式和visitor模式

##### 测试

同样使用Hello.g4语法文件，在IDEA中，打开Hello.g4右击"Test Rule"，ANTLR视图显示如下：


![image-20210422155519676.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7b9c28ceba347e8bdf867f44c3ef622~tplv-k3u1fbpfcp-watermark.image)

#### 代码实现

有了以上的测试就可以使用代码来获取Parse tree，进行遍历；看下面一个简单的实例：

```java
public class HelloDemo {
    public static void main(String[] args) {
        CharStream input = CharStreams.fromString("hello zhaohui");
        HelloLexer lexer = new HelloLexer(input);
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        HelloParser parser = new HelloParser(tokens);
        ParseTree tree = parser.r();
        System.out.println(tree.toStringTree(parser));
    }
}
```

输出结果如下：

```
(r hello zhaohui)
```



## 解析引擎

解析过程分为词法解析和语法解析。 词法解析器用于将 SQL拆解为不可再分的原子符号，称为Token。并根据不同数据库方言所提供的字典，将其归类为关键字，表达式，字面量和操作符。 再使用语法解析器将词法解析器的输出转换为抽象语法树。

从3.0.x 版本开始，使用ANTLR 来做词法解析器，每种支持的数据库都有自己的方言，针对每种数据库都有各自的解析器；通过上面的了解我们可以通过ANTLR来自动生成需要的解析器，前提是我们有Lexer和Parser文件；

### 语法文件

ANTLR在Github上提供了各种数据的语法文件，路径如下：

**文件路径**：[https://github.com/antlr/grammars-v4/tree/master/sql](web://urllink//https://github.com/antlr/grammars-v4/tree/master/sql/)

以Mysql为例，包含了两个文件：

```
MySqlLexer.g4
MySqlParser.g4
```

这样就可以通过相关工具生成需要的解析类了，在shardingsphere-sql-parser-mysql中可以发现自动生成类(autogen)：


![image-20210422170017851.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b48f35f14f384729bfb65367a679a249~tplv-k3u1fbpfcp-watermark.image)

当然我们也可以在IDEA中做一个简单的测试，输入一条常见的查询SQL：

```
SELECT * FROM ORDER WHERE USER_ID=111;
```

生成的树结构如下所示：


![parseTree.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91d0eb1b21b44ad09aa18171ef4ed5cc~tplv-k3u1fbpfcp-watermark.image)

### 解析引擎

ShardingSphere-JDBC提供的解析引擎类为：`SQLParserEngine`，主要的一个核心方法如下：

```java
private SQLStatement parse0(final String sql, final boolean useCache) {
        if (useCache) {
            Optional<SQLStatement> cachedSQLStatement = cache.getSQLStatement(sql);
            if (cachedSQLStatement.isPresent()) {
                return cachedSQLStatement.get();
            }
        }
        ParseTree parseTree = new SQLParserExecutor(databaseTypeName, sql).execute().getRootNode();
        SQLStatement result = (SQLStatement) ParseTreeVisitorFactory.newInstance(databaseTypeName, VisitorRule.valueOf(parseTree.getClass())).visit(parseTree);
        if (useCache) {
            cache.put(sql, result);
        }
        return result;
    }
```

两个参数分别是：逻辑SQL、是否使用缓存；返回值为SQLStatement；首先会进行是否使用缓存的判断，接下来就是关键的两步：逻辑SQL转换为ParseTree、访问ParseTree获取SQLStatement；

#### 转换ParseTree

要转换SQL为ParseTree，首先需要获取Parser，而获取Parser需要获取Lexer，写法其实和上面的`HelloDemo`差不多：

```java
private static SQLParser createSQLParser(final String sql, final SQLParserConfiguration configuration) {
        Lexer lexer = (Lexer) configuration.getLexerClass().getConstructor(CharStream.class).newInstance(CharStreams.fromString(sql));
        return configuration.getParserClass().getConstructor(TokenStream.class).newInstance(new CommonTokenStream(lexer));
    }
```

不同的数据类型会获取不同的Lexer和SQLParser；ShardingSphere-JDBC提供了多种数据库支持；

- Lexer：`MySQLLexer`，`OracleLexer`，`PostgreSQLLexer`，`SQL92Lexer`，`SQLServerLexer`；

- SQLParser：`MySqlParser`，`OracleParser`，`PostgreSQLParser`，`SQL92Parser`，`SQLServerParser`；

以上类其实都是对自动生成类的包装，以MysqlParser为例：

```java
public final class MySQLParser extends MySQLStatementParser implements SQLParser {
    
    public MySQLParser(final TokenStream input) {
        super(input);
    }
    
    @Override
    public ASTNode parse() {
        return new ParseASTNode(execute());
    }
}
```

执行MySQLParser的parser方法，其实调用的是自动生成类MySQLStatementParser中的execute方法；

#### 获取SQLStatement

有了ParseTree接下来就需要遍历树获取SQLStatement，ShardingSphere-JDBC默认使用的遍历方式是`visitor`方式；通过`visitor`对抽象语法树遍历构造域模型，通过域模型(`SQLStatement`)去提炼分片所需的上下文，并标记有可能需要改写的位置，同样每种数据库都要提供各自的`visitor`，目前支持的数据库包括：

visitor：`MySQLVisitor`，`OracleVisitor`，`PostgreSQLVisitor`，`SQL92Visitor`，`SQLServerVisitor`；

##### SQLStatement

通过`visitor`生成对应的`SQLStatement`，不同的SQL生成的SQLStatement是不同的，大体可以分为这么几类：

- DALStatement：全称**Data Access Layer**，数据库访问层，包括show databases、tables等；
- DMLStatement：全称**Data Manipulation Language**，数据库操作语言，包括增删改查等；
- DCLStatement：全称**Data Control Language**，数据库控制语言，包括授权，教授控制等；
- DDLStatement：全称**Data Definition Language**，数据库定义语言，包括创建、修改、删除表等；
- RLStatement：全称**Replication**，包括主从复制等；
- TCLStatement：全称**Transaction Control Language**，事务控制语言，包括设置保存点，回滚等；

打开对应数据库的语法文件，可以发现里面有对应的规则，如MySqlParser：

```
sqlStatement
    : ddlStatement | dmlStatement | transactionStatement
    | replicationStatement | preparedStatement
    | administrationStatement | utilityStatement
    ;
```

以上每种类型都提供了自己的`visitor`：

DALVisitor、DCLVisitor、DDLVisitor、DMLVisitor、RLVisitor、TCLVisitor

##### DMLStatement

以最常见的查询SQL为例，生成的是一个DMLStatement，常见的子类有：

DMLStatement：`CallStatement`、`DeleteStatement`、`DoStatement`、`InsertStatement`、`ReplaceStatement`、`SelectStatement`、`UpdateStatement`；

对应的语法文件也有对应关系：

```
dmlStatement
    : selectStatement | insertStatement | updateStatement
    | deleteStatement | replaceStatement | callStatement
    | loadDataStatement | loadXmlStatement | doStatement
    | handlerStatement
    ;
```

以上每种操作类型都需要在对应的`Visitor`中进行重载，以Mysql为例对应的DMLVisitor为`MySQLDMLVisitor`，相关select语句的方法重载，访问者模式遍历之后生成SelectStatement；

```java
 @Override
    public ASTNode visitSelect(final SelectContext ctx) {
        // TODO :Unsupported for withClause.
        SelectStatement result = (SelectStatement) visit(ctx.unionClause());
        result.setParameterCount(getCurrentParameterIndex());
        return result;
    }
    
    @SuppressWarnings("unchecked")
    @Override
    public ASTNode visitSelectClause(final SelectClauseContext ctx) {
        SelectStatement result = new SelectStatement();
        result.setProjections((ProjectionsSegment) visit(ctx.projections()));
        if (null != ctx.selectSpecification()) {
            result.getProjections().setDistinctRow(isDistinct(ctx));
        }
        if (null != ctx.fromClause()) {
            CollectionValue<TableReferenceSegment> tableReferences = (CollectionValue<TableReferenceSegment>) visit(ctx.fromClause());
            for (TableReferenceSegment each : tableReferences.getValue()) {
                result.getTableReferences().add(each);
            }
        }
        if (null != ctx.whereClause()) {
            result.setWhere((WhereSegment) visit(ctx.whereClause()));
        }
        if (null != ctx.groupByClause()) {
            result.setGroupBy((GroupBySegment) visit(ctx.groupByClause()));
        }
        if (null != ctx.orderByClause()) {
            result.setOrderBy((OrderBySegment) visit(ctx.orderByClause()));
        }
        if (null != ctx.limitClause()) {
            result.setLimit((LimitSegment) visit(ctx.limitClause()));
        }
        if (null != ctx.lockClause()) {
            result.setLock((LockSegment) visit(ctx.lockClause()));
        }
        return result;
    }
```

##### SelectStatement

查询SQL对应`SelectStatement`，部分代码如下：

```java
public final class SelectStatement extends DMLStatement {
    
    private ProjectionsSegment projections;
    private final Collection<TableReferenceSegment> tableReferences = new LinkedList<>();
    private WhereSegment where;
    private GroupBySegment groupBy;
    private OrderBySegment orderBy;
    private LimitSegment limit;
    private SelectStatement parentStatement;
    private LockSegment lock;
}
```

可以发现里面包含了很多`Segment`，每个`Segment`其实就是整个SQL的一部分，上面这些关键字是不是都很熟悉，都是在查询语句中会出现的；其他类型这里就不贴代码了，根据每种类型生成各自的`Segment`；最后将`SQLStatement`包装成上下文`SQLStatementContext`给下游的路由引擎使用；

同样语法文件也有对应关系：

```
selectStatement
    : querySpecification lockClause?                                #simpleSelect
    | queryExpression lockClause?                                   #parenthesisSelect
    | querySpecificationNointo unionStatement+
        (
          UNION unionType=(ALL | DISTINCT)?
          (querySpecification | queryExpression)
        )?
        orderByClause? limitClause? lockClause?                     #unionSelect
    | queryExpressionNointo unionParenthesis+
        (
          UNION unionType=(ALL | DISTINCT)?
          queryExpression
        )?
        orderByClause? limitClause? lockClause?                     #unionParenthesisSelect
    ;
```



## 总结

本文重点介绍了整个分片流程中的解析流程，整个解析的核心就是ANTLR，如果了解了ANTLR的相关语法，以及遍历方式，那解析引擎基本没什么难度了，ANTLR官方文档还是比较全面的，有兴趣的可以去细读；下文继续分析分片的路由机制。

## 参考

https://shardingsphere.apache.org/document/current/cn/overview/

## 感谢关注
> 可以关注微信公众号「**回滚吧代码**」，第一时间阅读，文章持续更新；专注Java源码、架构、算法和面试。

