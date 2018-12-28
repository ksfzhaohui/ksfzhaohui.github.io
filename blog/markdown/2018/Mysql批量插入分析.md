**前言**  
最近发现几个项目中都有批次插入数据库的功能，每个项目中批次插入的写法有一些差别，所以本文打算对Mysql的批次插入做一个详细的分析。

**准备**  
1.jdk1.7，mysql5.6.38  
2.准备库和表

```sql
create database db3;
 
CREATE TABLE `travelrecord` (
  `id` int(11) NOT NULL,
  `name` varchar(255) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**测试与分析**  
下面准备几种插入的方式来分析优劣：  
1.Statement插入方式

```java
public class JDBCBatch2 {
 
    public static void main(String[] args) {
        String url = "jdbc:mysql://192.168.237.128:3306/db3";
        String username = "root";
        String password = "root";
        Connection con = null;
        try {
            con = DriverManager.getConnection(url, username, password);
            Statement statemenet = (Statement) con.createStatement();
            con.setAutoCommit(false);
            List<Travelrecord> list = new ArrayList<Travelrecord>();
            for (int j = 0; j < 100; j++) {
                list.add(new Travelrecord(j + 1, "hsfhsdhfsdhfhsdhfhsdfhsdhfhsdhfhsdhfhsd"));
            }
            long startTime = System.currentTimeMillis();
            for (Travelrecord t : list) {
                statemenet.execute("insert into travelrecord (id,name) values (" + t.getId() + ",\"" + t.getName() + "\")");
            }
            long endTime = System.currentTimeMillis();
            con.commit();
            System.out.println("times=" + (endTime - startTime));
        } catch (Exception se) {
            if (con != null) {
                try {
                    System.out.println("rollback");
                    con.rollback();
                } catch (SQLException e) {
                    System.err.println("rollback error");
                }
            }
            se.printStackTrace();
        }
    }
}
```

准备数据，然后通过Statement方式插入数据，插入10000条数据大概在6秒多左右，同时可以监控服务器数据包;  
监控命令：

```bash
tcpdump -i any port 3306
```

日志如下:

```bash
23:51:06.590979 IP 192.168.237.1.54031 > 192.168.237.128.mysql: Flags [P.], seq 13601:13635, ack 9361, win 255, length 34
23:51:06.591056 IP 192.168.237.128.mysql > 192.168.237.1.54031: Flags [P.], seq 9361:9438, ack 13635, win 266, length 77
23:51:06.591238 IP 192.168.237.1.54031 > 192.168.237.128.mysql: Flags [P.], seq 13635:13728, ack 9438, win 254, length 92
23:51:06.591342 IP 192.168.237.128.mysql > 192.168.237.1.54031: Flags [P.], seq 9438:9449, ack 13728, win 266, length 11

```

以上截取了其中一条插入语句的数据包日志，详细的数据包可以通过如下命令监控：

```
tcpdump -i any port 3306 -X
```

详细日志：

```
23:55:36.791580 IP 192.168.237.1.54315 > 192.168.237.128.mysql: Flags [P.], seq 1418:1452, ack 913, win 253, length 34
    0x0000:  4500 004a 7d67 4000 8006 2173 c0a8 ed01  E..J}g@...!s....
    0x0010:  c0a8 ed80 d42b 0cea f108 592e 9348 0672  .....+....Y..H.r
    0x0020:  5018 00fd b524 0000 1e00 0000 0373 656c  P....$.......sel
    0x0030:  6563 7420 4040 7365 7373 696f 6e2e 7478  ect.@@session.tx
    0x0040:  5f72 6561 645f 6f6e 6c79 0000 0000 0000  _read_only......
    0x0050:  0000 0000 0000 0000 0000                 ..........
23:55:36.791897 IP 192.168.237.128.mysql > 192.168.237.1.54315: Flags [P.], seq 913:990, ack 1452, win 266, length 77
......
23:55:36.795176 IP 192.168.237.1.54315 > 192.168.237.128.mysql: Flags [P.], seq 1452:1544, ack 990, win 252, length 92
    0x0000:  4500 0084 7d69 4000 8006 2137 c0a8 ed01  E...}i@...!7....
    0x0010:  c0a8 ed80 d42b 0cea f108 5950 9348 06bf  .....+....YP.H..
    0x0020:  5018 00fc 1daa 0000 5800 0000 0369 6e73  P.......X....ins
    0x0030:  6572 7420 696e 746f 2074 7261 7665 6c72  ert.into.travelr
    0x0040:  6563 6f72 6420 2869 642c 6e61 6d65 2920  ecord.(id,name).
    0x0050:  7661 6c75 6573 2028 312c 2268 7366 6873  values.(1,"hsfhs
    0x0060:  6468 6673 6468 6668 7364 6866 6873 6466  dhfsdhfhsdhfhsdf
    0x0070:  6873 6468 6668 7364 6866 6873 6468 6668  hsdhfhsdhfhsdhfh
    0x0080:  7364 2229 0000 0000 0000 0000 0000 0000  sd")............
    0x0090:  0000 0000                           
23:55:36.796093 IP 192.168.237.128.mysql > 192.168.237.1.54315: Flags [P.], seq 990:1001, ack 1544, win 266, length 11

```

可以发现每个sql语句包前面都有一个_select.@@session.tx\_read\_only_包，这是因为mysql jdbc驱动设置_useLocalSessionState_=false，每一次都需要检测目标数据库isReadOnly的状态，  
所以每次都发送select.@@session.tx\_read\_only包，可以设置useLocalSessionState=true使用连接对象本地的状态，可以修改url如下：

```
jdbc:mysql://192.168.237.128:3306/db3?useLocalSessionState=true
```

再次运行，观察日志：

```
00:11:32.342852 IP 192.168.237.1.54949 > 192.168.237.128.mysql: Flags [P.], seq 1857:1949, ack 957, win 252, length 92
00:11:32.343076 IP 192.168.237.128.mysql > 192.168.237.1.54949: Flags [P.], seq 957:968, ack 1949, win 266, length 11

```

日志中省掉了select.@@session.tx\_read\_only的过程，提升插入的性能，具体代码可以参考ConnectionImpl的isReadOnly方法：

```
public boolean isReadOnly(boolean useSessionStatus) throws SQLException {
        if (useSessionStatus && !this.isClosed && versionMeetsMinimum(5, 6, 5) && !getUseLocalSessionState()) {
            java.sql.Statement stmt = null;
            java.sql.ResultSet rs = null;
 
            try {
                try {
                    stmt = getMetadataSafeStatement();
 
                    rs = stmt.executeQuery("select @@session.tx_read_only");
                    if (rs.next()) {
                        return rs.getInt(1) != 0; // mysql has a habit of tri+ state booleans
                    }
                    ......
```

2.PreparedStatement方式

```java
public class JDBCBatch {
 
    public static void main(String[] args) {
        String url = "jdbc:mysql://192.168.237.128:3306/db3";
        String username = "root";
        String password = "root";
        String sql = "insert into travelrecord (id,name) values (?,?)";
        Connection con = null;
        try {
            con = DriverManager.getConnection(url, username, password);
            PreparedStatement pstmt = con.prepareStatement(sql);
            con.setAutoCommit(false);
            List<Travelrecord> list = new ArrayList<Travelrecord>();
            for (int j = 0; j < 100; j++) {
                list.add(new Travelrecord(j + 1, "hsfhsdhfsdhfhsdhfhsdfhsdhfhsdhfhsdhfhsd"));
            }
 
            long startTime = System.currentTimeMillis();
            for (Travelrecord t : list) {
                pstmt.setInt(1, t.getId());
                pstmt.setString(2, t.getName());
                pstmt.addBatch();
            }
            pstmt.executeBatch();
            long endTime = System.currentTimeMillis();
            con.commit();
            System.out.println("times=" + (endTime - startTime));
        } catch (Exception se) {
            if (con != null) {
                try {
                    System.out.println("rollback");
                    con.rollback();
                } catch (SQLException e) {
                    System.err.println("rollback error");
                }
            }
            se.printStackTrace();
        }
    }
}
```

PreparedStatement比起Statement有很多优势，其中一条就是PreparedStatement比Statement更快，SQL语句会预编译在数据库系统中，执行计划同样会被缓存起来，它允许数据库做参数化查询。同样插入10000条数据，时间大概在5秒多左右，比起Statement有一定优势，但是不明显；PreparedStatement使用的是批次提交，速度不应该这么查，同样观察日志：

```
00:23:57.679444 IP 192.168.237.1.62510 > 192.168.237.128.mysql: Flags [P.], seq 2460:2494, ack 1694, win 255, length 34
00:23:57.679617 IP 192.168.237.128.mysql > 192.168.237.1.62510: Flags [P.], seq 1694:1771, ack 2494, win 266, length 77
00:23:57.680139 IP 192.168.237.1.62510 > 192.168.237.128.mysql: Flags [P.], seq 2494:2586, ack 1771, win 255, length 92
00:23:57.680349 IP 192.168.237.128.mysql > 192.168.237.1.62510: Flags [P.], seq 1771:1782, ack 2586, win 266, length 11

```

发现和Statement没有区别，一条语句对应了一个包，没有批次的效果，查看PreparedStatement的executeBatch方法，部分代码如下：

```java
try {
    statementBegins();
    clearWarnings();
    if (!this.batchHasPlainStatements && this.connection.getRewriteBatchedStatements()) {           
        if (canRewriteAsMultiValueInsertAtSqlLevel()) {
            return executeBatchedInserts(batchTimeout);
        }   
        if (this.connection.versionMeetsMinimum(4, 1, 0) 
                    && !this.batchHasPlainStatements
                    && this.batchedArgs != null
                    && this.batchedArgs.size() > 3 /* cost of option setting rt-wise */) {
            return executePreparedBatchAsMultiStatement(batchTimeout);
        }
    }
    return executeBatchSerially(batchTimeout);
} finally {
    this.statementExecuting.set(false);         
    clearBatch();
}
```

其中大致逻辑就是如果canRewriteAsMultiValueInsertAtSqlLevel()为true，那么执行批次插入(executeBatchedInserts)，否则执行串联插入(executeBatchSerially)；具体可以通过url上添加参数_rewriteBatchedStatements_：

```
jdbc:mysql://192.168.237.128:3306/db3?rewriteBatchedStatements=true
```

再次运行，插入10000条数据只需要100ms左右，观察日志：

```
00:37:35.489633 IP 192.168.237.1.63528 > 192.168.237.128.mysql: Flags [P.], seq 212694:263794, ack 1069, win 252, length 51100
00:37:35.489670 IP 192.168.237.128.mysql > 192.168.237.1.63528: Flags [.], ack 263794, win 2730, length 0
00:37:35.489716 IP 192.168.237.1.63528 > 192.168.237.128.mysql: Flags [.], seq 263794:279854, ack 1069, win 252, length 16060
00:37:35.489849 IP 192.168.237.128.mysql > 192.168.237.1.63528: Flags [.], ack 279854, win 2777, length 0

```

可以发现数据包不是原来的92个字节了，每个包的大小大幅度提升，具体分多少次提交，每次提交多少数据量，可以查看PreparedStatement的computeBatchSize方法：

```java
protected int computeBatchSize(int numBatchedArgs) throws SQLException {
    synchronized (checkClosed().getConnectionMutex()) {
        long[] combinedValues = computeMaxParameterSetSizeAndBatchSize(numBatchedArgs);
         
        long maxSizeOfParameterSet = combinedValues[0];
        long sizeOfEntireBatch = combinedValues[1];
         
        int maxAllowedPacket = this.connection.getMaxAllowedPacket();
         
        if (sizeOfEntireBatch < maxAllowedPacket - this.originalSql.length()) {
            return numBatchedArgs;
        }
         
        return (int)Math.max(1, (maxAllowedPacket - this.originalSql.length()) / maxSizeOfParameterSet);
    }
}
```

此方法计算每次提交批量数据中的多少条数据，其中一个maxAllowedPacket参数，此参数在服务器端配置用来限制客户端每个包的最大字节数；  
查询maxAllowedPacket：

```sql
mysql> show VARIABLES like '%max_allowed_packet%';
+--------------------------+------------+
| Variable_name            | Value      |
+--------------------------+------------+
| max_allowed_packet       | 4194304    |
| slave_max_allowed_packet | 1073741824 |
+--------------------------+------------+
```

设置maxAllowedPacket：

```sql
mysql> set global max_allowed_packet =1024*10;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

此方式可以很好的执行批量数据的插入，但是如果数据量很大，一下执行所有数据的批次插入，很容易造成客户端内存的溢出，所以也可以使用第三种方式；

3.PreparedStatement分批次方式  
部分代码如下：

```java
for (int i = 0; i < 10; i++) {
    for (int k = 0; k < 1000; k++) {
        pstmt.setInt(1, list.get(i * 1000 + k).getId());
        pstmt.setString(2, list.get(i * 1000 + k).getName());
        pstmt.addBatch();
    }
    pstmt.executeBatch();
}
```

同样是插入10000条数据，但是这种方式是，分10次批次插入数据，有效的控制了内存的消耗，可以做一个简单的实验；

设置启动参数

```
-Xms5000k -Xmx5000k
```

然后分别使用第二种方式和第三种方式插入10w条数据，第二种方式直接内存溢出，而第三种方式可以完整的将数据插入；当然分批次插入肯定比一次性插入速度慢，所以可以在内存和速度方面做一个简单的权衡。

**总结**  
本文通过三种方式来插入数据，从而了解Mysql批次插入的过程，了解到useLocalSessionState和rewriteBatchedStatements参数对性能的影响，以及maxAllowedPacket对数据包的大小限制；最后建议要在内存和速度方面做一个权衡。