---
layout: post
title: Kudu + Impala的介绍
category: mysql
tags: [mysql]
keywords: impala,kudu
excerpt: Impala和Kudu做了深度集成，是Cloudera公司主推的新式大数据解决方案，PB级别大数据查询解决方案
lock: noneed
---

## 1、概述

### 背景

Kudu和Impala均是Cloudera贡献给Apache基金会的顶级项目。Kudu作为底层存储，在支持高并发低延迟kv查询的同时，还保持良好的Scan性能，该特性使得其理论上能够同时兼顾OLTP类和OLAP类查询。Impala作为老牌的SQL解析引擎，其面对即席查询(Ad-Hoc  Query)类请求的稳定性和速度在工业界得到过广泛的验证，Impala并没有自己的存储引擎，其负责解析SQL，并连接其底层的存储引擎。在发布之初Impala主要支持HDFS，Kudu发布之后，Impala和Kudu更是做了深度集成。

**比较Hive**

Impala定位类似Hive

- Impala更关注即席查询SQL的快速解析，对于执行时间过长的SQL，仍旧是Hive更合适。
- 对于GroupBy等SQL查询，Impala进行的是内存计算，所以Impala对机器配置要求较高，官方建议内存128G以上。Hive底层对应的是传统的MapReduce计算框架，虽然执行效率低，但是稳定性好，对机器配置要求也低。

执行效率是Impala的最大优势，对于存储在HDFS中的数据，Impala的解析速度本来就远快于Hive，有了Kudu加成之后，更是如虎添翼，部分查询执行速度差别可达百倍。

> Kudu和Impala的英文原意是来自非洲的两个不同品种的羚羊，Cloudera这个公司非常喜欢用跑的快的动物来作为其产品的命名。

### OLAP与OLTP

**OLTP(On-line Transaction Processing)** 面向的是高并发低延时的增删改查(INSERT, DELETE, UPDATE, SELECT, etc..)。

**OLAP(On-line Analytical Processing)** 面向的是BI分析型数据请求，其对延时有较高的容忍度，处理数据量相较OLTP要大很多。

传统意义上与OLTP对应的是MySQL等关系型数据库，与OLAP相对应的则是数据仓库。OLTP与OLAP所面向的数据存储查询引擎是不同的，其处理的请求不一样，所需要的架构也大不相同。这个特点意味着数据需要储存在至少两个地方，需要定期或者实时的同步，同时还需要保持一致性，该特点对数据开发工程师造成了极大的困扰，浪费了同学们大量的时间在数据的同步和校验上。Kudu+Impala的出现，虽然不能完美的解决这个问题，但不可否认，其缓解了这个矛盾。

在此需要注意的是OLTP并没有严格要求其事务处理满足ACID四个条件。事实上，OLTP是比ACID更早出现的概念。在本文中，OLTP和OLAP的概念关注在数据量、并发量、延时要求等方面，不关注事务。

### 术语

- **HDFS**是Hadoop生态圈最基础的存储引擎，请注意HDFS的设计主要为大文件存储，为高吞吐量的读取和写入服务，HDFS不适合存储小文件，也不支持大量的随机读写。
- **MapReduce**是分布式计算最基础的计算框架，通过把任务分解为多个Mapper和Reducer，该计算框架可以很好地处理常见的大数据任务。
- **Hadoop**最早由HDFS+MapReduce构成，随着Hadoop2.0以及3.0的发布，Hadoop正在被赋予更多的功能。
- **Hbase**启发于Google的Bigtable论文，项目最早开始于Powerset公司。Hbase建立在HDFS之上，拥有极高的随机读写性能，是典型的OLTP类请求处理引擎。与此同时，Hbase也拥有不错的Scan性能，可以处理部分类型的OLAP类请求。
- **Spark**集迭代式计算、流式计算于一身的新一代计算引擎，相比MapReduce，Spark提供了更丰富的分布式计算原语，可以更高效的完成分布式计算任务。
- **Ad-Hoc Query**通常翻译为**即席查询**，是数据仓库中的一个重要概念。数据探索和分析应用中通常会任意拼凑临时SQL，并且对查询速度有一定要求，该类查询统称为即席查询。
- **列式存储**相比于**行式存储**，列式存储把相同列的数据放在一起。因为相同列的数据重复度更高，所以在存储上可以提供更高的压缩比。又因为大部分BI分析只读取部分列，相比行式存储，列式存储只需要扫描需要的列，读取的数据更少，因此可以提供更快速的查询。常见的列式存储协议有Parquet等。

## 2、Kudu的介绍

### kudu是什么

Kudu最早是Cloudera公司开发，并与2015年12月3日贡献给Apache基金会，[2016年7月25日正式宣布毕业](https://blogs.apache.org/foundation/entry/apache_software_foundation_announces_apache)，升级为 Apache 顶级项目。值得注意的是Kudu在开发之中得到了中国公司小米的大力支持，小米深度参与到了Kudu的开发之中，拥有一位Kudu的Committer。

Kudu是围绕Hadoop生态圈建立存储引擎，Kudu拥有和Hadoop生态圈共同的设计理念，它运行在普通的服务器上、可分布式规模化部署、并且满足工业界的高可用要求。其设计理念为**fast analytics on fast data.**。Kudu的大部分场景和Hbase类似，其设计降低了随机读写性能，提高了扫描性能，在大部分场景下，Kudu在拥有接近Hbase的随机读写性能的同时，还有远超Hbase的扫描性能。

区别于Hbase等存储引擎，Kudu有如下优势：

- 快速的OLAP类查询处理速度
- 与MapReduce、Spark等Hadoop生态圈常见系统高度兼容，其连接驱动由官方支持维护
- 与Impala深度集成，相比HDFS+Parquet+Impala的传统架构，Kudu+Impala在绝大多数场景下拥有更好的性能。
- 强大而灵活的一致性模型，允许用户对每个请求单独定义一致性模型，甚至包括强序列一致性。
- 能够同时支持OLTP和OLAP请求，并且拥有良好的性能。
- Kudu集成在ClouderaManager之中，对运维友好。
- 高可用。采用Raft Consensus算法来作为master失败后选举模型，即使选举失败，数据仍然是可读的。
- 支持结构化的数据，纯粹的列式存储，省空间的同时，提供更高效的查询速度。

### 典型使用场景

- 流式实时计算场景

  流式计算场景通常有持续不断地大量写入，与此同时这些数据还要支持近乎实时的读、写以及更新操作。Kudu的设计能够很好的处理此场景。

- 时间序列存储引擎(TSDB)

  Kudu的hash分片设计能够很好地避免TSDB类请求的局部热点问题。同时高效的Scan性能让Kudu能够比Hbase更好的支持查询操作。

- 机器学习&数据挖掘

  机器学习和数据挖掘的中间结果往往需要高吞吐量的批量写入和读取，同时会有少量的随机读写操作。Kudu的设计可以很好地满足这些中间结果的存储需求

- 与历史遗产数据共存

  在工业界实际生产环境中，往往有大量的历史遗产数据。Impala可以同时支持HDFS、Kudu等多个底层存储引擎，这个特性使得在使用的Kudu的同时，不必把所有的数据都迁移到Kudu。

### 重要概念

- 列式存储

  毫无疑问，Kudu是一个纯粹的列式存储引擎，相比Hbase只是按列存放数据，Kudu的列式存储更接近于Parquet，在支持更高效Scan操作的同时，还占用更小的存储空间，列式存储有如此优势，

  主要因为两点：

  1. 通常意义下的OLAP查询只访问部分列数据，列存储引擎在这种情况下支持按需访问，而行存储引擎则必须检索一行中的所有数据。

  2.  数据按列放一起一般意义来讲会拥有更高的压缩比，这是因为列相同的数据往往拥有更高的相似性。

- Table

  Kudu中所有的数据均存储在Table之中，每张表有其对应的表结构和主键，数据按主键有序存储。因为Kudu设计为支持超大规模数据量，Table之中的数据会被分割成为片段，称之为Tablet。

- Tablet

  一个Tablet把相邻的数据放在一起，跟其他分布式存储服务类似，一个Tablet会有多个副本放置在不同的服务器上面，在同一时刻，仅有一个Tablet作为leader存在，每个副本均可单独提供读操作，写操作则需要一致性同步写入。

- Tablet Server

  Tablet服务顾名思义，对Tablet的读写操作会通过该服务完成。对于一个给定的tablet，有一个作为leader，其他的作为follower，leader选举和灾备原则遵循<mark>Raft</mark>一致性算法，该算法在后文中有介绍。需要注意的是一个Tablet服务所能承载的Tablet数量有限，这也要求的Kudu表结构的设计需要合理的设置Partition数量，太少会导致性能降低，太多会造成过多的Tablet,给Tablet服务造成压力。

- Master

  master存储了其他服务的所有元信息，在同一时刻，最多有一个master作为leader提供服务，leader宕机之后会按照Raft一致性算法进行重新选举。

  master会协调client传来的元信息读写操作。比如当创建一个新表的时候，client发送请求给master，master会转发请求给catelog、 tablet等服务。

  **master本身并不存储数据，数据存储在一个tablet之中，并且会按照正常的tablet进行副本备份。**

  Tablet服务会每秒钟跟master进行心跳连接。

- Raft Consensus Algorithm 一致性算法

  Kudu  使用Raft一致性算法，该算法将节点分为follower、candidate、leader三种角色，当leader节点宕机时，follower会成为candidate并且通过多数选举原则成为一个新的leader，因为有多数选举原则，所以在任意时刻，最多有一个leader角色。leader接收client上传的数据修改指令并且分发给follower，当多数follower写入时，leader会认为写入成功并告知client。

- Catalog Table

  Catelog表存储了Kudu的一些元数据，包括Tables和Tablets。

### kudu的架构图

![](\assets\images\2021\mysql\kudu-arch.png)

从上图可以看出有三台Master，其中一个是leader，另外两个是follower。

有四台Tablet server，n个tablets及其副本均匀分布在这四台机器上。每个tablet有一个leader，两个follower。每个表会按照分片的数量分成多个tablet。

### Kudu的缺点

1、主键的限制

- 表创建后主键不可更改；
- 一行对应的主键内容不可以被Update操作修改。要修改一行的主键值，需要删除并新增一行新数据，并且该操作无法保持原子性；
- 主键的类型不支持DOUBLE、FLOAT、BOOL，并且主键必须是非空的(NOT NULL)；
- 自动生成的主键是不支持的；
- 每行对应的主键存储单元(CELL)最大为16KB。

2、列的限制

- MySQL中的部分数据类型，如DECIMAL, CHAR, VARCHAR, DATE, ARRAY等不支持；
- 数据类型以及是否可为空等列属性不支持修改；
- 一张表最多有300列。

3、表的限制

- 表的备份数必须为奇数，最大为7；
- 备份数在设置后不可修改。

4、单元Cells的限制

- 单元对应的数据最大为64KB，并且是在压缩前

5、分片的限制

- 分片只支持手动指定，自动分片不支持；
- 分片设定不支持修改，修改分片设定需要”建新表-导数据-删老表”操作；
- 丢掉多数备份的Tablets需要手动修复。

6、容量的限制

- 建议tablet servers的最大数量为100；
- 建议masters的最大数量为3；
- 建议每个tablet server存储的数据最大为4T（此处存疑，为何会有4T这么小的限制？）；
- 每个tablet server存储的tablets数量建议在1000以内；
- 每个表分片后的tablets存储在单个tablet server的最大数量为60。



## 3、Impala的介绍

### Impala是什么

Impala是建立在Hadoop生态圈的交互式SQL解析引擎，Impala的SQL语法与Hive高度兼容，并且提供标准的ODBC和JDBC接口。Impala本身不提供数据的存储服务，其底层数据可来自HDFS、Kudu、Hbase甚至亚马逊S3。

Impapa最早由Cloudera公司开发，于15年12月贡献给Apache基金会，目前其正式名字为Apache Impala(incubating)

Impala本身并不是Hive的完全替代品，对于一些大吞吐量长时间执行的请求，Hive仍然是最稳定最佳的选择，哪怕是SparkSQL，其稳定性也无法跟Hive媲美。

稳定性方面Impala不如Hive，但是在执行效率方面，Impala毫无疑问可以秒杀Hive。Impala采用内存计算模型，对于分布式Shuffle，可以尽可能的利用现代计算机的内存和CPU资源。同时，Impala也有预处理和分析技术，表数据插入之后可以用**COMPUTE STATS**指令来让Impala对行列数据深度分析。

### 优势

- 和Hive高度相似的SQL语法，无需太多学习成本
- 超大数据规模SQL解析的能力，高效利用内存与CPU利用，快速返回SQL查询结果。
- 集成多个底层数据源，HDFS、Kudu、Hbase等数据皆可通过Impala共享，并且无需进行数据同步。
- 与Hue深度集成，提供可视化的SQL操作以及work flow。
- 提供标准JDBC和ODBC接口，方便下游业务方无缝接入。
- 提供最多细化到列的权限管理，满足实际生产环境数据安全要求。

### 与Hive的兼容

Impala高度兼容Hive，不过有部分Hive的SQL特性在Impala中并不支持，其中包括：

- Data等类型不支持
- XML和Json函数不支持
- 多个DISTINCT不支持，完成多个DISTINCT需要如下操作

```sql
select v1.c1 result1, v2.c1 result2 from (select count(distinct col1) as c1 from t1) v1 cross join (select count(distinct col2) as c1  from t1) v2;
```

Impala和Hive的兼容不仅仅体现在语法上，在架构上Impala和Hive也保持着相当程度上的兼容性，Impala直接采用Hive的元数据库，对于公司而言，已经在Hive中的表结构无需迁移，Impala可以直接使用。

### Impala集成Kudu

Kudu+Impala为实时数据仓库存储提供了良好的解决方案。这套架构在支持随机读写的同时还能保持良好的Scan性能，同时其对Spark等流式计算框架有官方的客户端支持。这些特性意味着数据可以从Spark实时计算中实时的写入Kudu，上层的Impala提供BI分析SQL查询，对于数据挖掘和算法等需求可以在Spark迭代计算框架上直接操作Kudu底层数据。

### Impala的缺点

- Impala不适合超长时间的SQL请求；
- Impala不支持高并发读写操作，即使Kudu是支持的；
- Impala和Hive有部分语法不兼容。

### Springboot连接Impala

新建springboot项目，pom.xml导入

```xml
<dependency>
  <groupId>com.cloudera.impala</groupId>
  <artifactId>ImpalaJDBC41</artifactId>
  <version>2.6.4</version>
</dependency>
<dependency>
  <groupId>com.cloudera.impala</groupId>
  <artifactId>libthrift</artifactId>
  <version>0.9.0</version>
</dependency>
<dependency>
  <groupId>com.cloudera.impala</groupId>
  <artifactId>hive-service</artifactId>
  <version>1.0.0</version>
</dependency>
```

ImpalaJDBC41.jar包里面不能带有slfj4.jar日志jar包，否则会有springboot自带日志jar包冲突（相同的类路径下有两个slfj4.jar包）导致启动失败

> 直接连接

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
 
public class Impala_jdbc {
    private static String DRIVER = "com.cloudera.impala.jdbc41.Driver";
    private static String URL = "jdbc:impala://IP:21050/数据库名";
 
    public static void main(String[] args)
    {
        Connection conn = null;
        ResultSet rs = null;
        PreparedStatement pst = null;
 
        try {
            Class.forName(DRIVER);
            conn = DriverManager.getConnection(URL);
            pst = conn.prepareStatement("select * from 数据库名.表名 limit 3");
            rs = pst.executeQuery();
            while (rs.next()) {
                //rs.get类型(字段列)：字段列从1开始算起
                System.out.println(rs.getString(1) + "," + rs.getObject(2) + "," + rs.getObject(3));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                conn.close();
                pst.close();
            } catch (SQLException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
}
```

> 使用连接池

pom.xml

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.1.16</version>
</dependency>
<dependency>
  <groupId>commons-dbutils</groupId>
  <artifactId>commons-dbutils</artifactId>
  <version>1.7</version>
</dependency>
<dependency>
  <groupId>com.cloudera</groupId>
  <artifactId>ImpalaJDBC41</artifactId>
  <version>2.6.3</version>
</dependency>
```

新建配置文件\src\main\resources\Impaladruid.properties

```properties
driverClassName=com.cloudera.impala.jdbc41.Driver
url=jdbc:impala://IP:21050/数据库名
 
initialSize=10
maxActive=100
maxWait=60000
 
timeBetweenEvictionRunsMillis=60000
minEvictableIdleTimeMillis=300000
validationQuery=SELECT 1
testWhileIdle=true
testOnBorrow=false
testOnReturn=false
poolPreparedStatements=false
maxPoolPreparedStatementPerConnectionSize=200
```

工具类

```java
import com.alibaba.druid.pool.DruidDataSourceFactory;
import org.apache.log4j.LogManager;
import org.apache.log4j.Logger; 
import javax.sql.DataSource;
import java.io.IOException;
import java.io.InputStream;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Properties;
 
public class MysqlUtil
{
    private static DataSource dataSource; //Druid 连接池
    private static Connection conn; //数据库连接对象
    private static Logger logger = LogManager.getLogger(MysqlUtil.class);
    private static InputStream in = null;
    private final static MysqlUtil mysqlUtil = new MysqlUtil();
    static
    {
        //使用druid.properties属性文件的配置方式 设置参数，文件名称没有规定但是属性文件中的key要一定的
        // 从druid.properties属性文件中获取key参数对应的value配置信息
        Properties properties = new Properties();
        try
        {
            //resources目录下的配置文件实际都会被编译进到 \target\classes 目录下
            in = mysqlUtil.getClass().getResourceAsStream("/Impaladruid.properties");
            properties.load(in);
            in.close();
        }
        catch (IOException e)
        {
            logger.error("ERROR:",e);//错误异常完整写入日志文件
//            e.printStackTrace();//窗口也打印错误信息
        }
        // 创建 Druid 连接池
        try {
            dataSource = DruidDataSourceFactory.createDataSource(properties);
        } catch (Exception e) {
            logger.error("ERROR:",e);//错误异常完整写入日志文件
//            e.printStackTrace();//窗口也打印错误信息
        }
 
        //从 连接池中获取一个 数据库连接对象
//        Connection  数据库连接对象conn  =  dataSource.getConnection();
        //调用数据库连接对象的close()方法，把连接对象归还给连接池，并不是关闭连接
//        conn.close();
    }
 
    //获得连接池
    public static DataSource getDataSource()
    {
        return dataSource;
    }
 
    //从 连接池中获取一个 数据库连接对象
    public static Connection getConnection()
    {
        //从 连接池中获取一个 数据库连接对象
        try {
//            System.out.println("从 连接池中获取一个 数据库连接对象");
            conn  =  dataSource.getConnection();
        } catch (Exception e) {
//            e.printStackTrace();//窗口也打印错误信息
            logger.error("ERROR:",e);//错误异常完整写入日志文件
        }
        return conn;
    }
 
    /*
        new QueryRunner(MysqlUtil.getDataSource())
                QueryRunner中传入连接池，交由QueryRunner自动操作连接池中的连接。提供了自定事务处理、自动释放资源等操作，无需再手动。
                所以无需手动调用conn.close()，交由QueryRunner自动管理。
    */
    //调用数据库连接对象的close()方法，把指定的连接对象归还给连接池，并不是关闭连接
    public static void connectionClose(Connection conn)
    {
        //调用数据库连接对象的close()方法，把连接对象归还给连接池，并不是关闭连接
        try {
            conn.close();
        } catch (SQLException e) {
//            e.printStackTrace();//窗口也打印错误信息
            logger.error("ERROR:",e);//错误异常完整写入日志文件
        }
    }
}
```

测试

```java
import org.apache.commons.dbutils.QueryRunner;
import org.apache.commons.dbutils.handlers.ArrayListHandler;
import org.apache.log4j.LogManager;
import org.apache.log4j.Logger;
import java.sql.*;
import java.util.List;
 
public class ImpalaTest{
     public static void main(String[] args){
        QueryRunner queryRunner = new QueryRunner(MysqlUtil.getDataSource());
        List<Object[]> arrayListResult = null;
        String sql =  "";
        try
        {
            arrayListResult = queryRunner.query(sql, new ArrayListHandler());
        } 
        catch (SQLException e) 
        {
            e.printStackTrace();
        }
        for (Object[] o : arrayListResult1)
        {
            System.out.println(o[0].toString());
        }
    }
}
```







参考：[https://blog.csdn.net/zimiao552147572/article/details/90234974](https://blog.csdn.net/zimiao552147572/article/details/90234974)

## 4、总结

1) Impala支持高并发读写吗？

不支持。虽然Impala设计为BI-即席查询平台，但是其单个SQL执行代价较高，不支持低延时、高并发场景。

2) Impala能代替Hive吗？

不能，Impala设计为内存计算模型，其执行效率高，但是稳定性不如Hive，对于长时间执行的SQL请求，Hive仍然是第一选择。

3) Impala需要多少内存？

类似于Spark，Impala会把数据尽可能的放入内存之中进行计算，虽然内存不够时，Impala会借助磁盘进行计算，但是毫无疑问，内存的大小决定了Impala的执行效率和稳定性。Impala官方建议内存要至少128G以上，并且把80%内存分配给Impala

4) Impala有Cache吗？

Impala不会对表数据Cache，Impala仅仅会Cache一些表结构等元数据。虽然在实际情况下，同样的query第二次跑可能会更快，但这不是Impala的Cache，这是Linux系统或者底层存储的Cache。

5) Impala可以添加自定义函数吗？

可以。Impala1.2版本支持的UDFs，不过Impala的UDF添加要比Hive复杂一些。

6) Impala为什么会这么快？

Impala为速度而生，其在执行效率细节上做了很多优化。在大的方面，相比Hive，Impala并没有采用MapReduce作为计算模型，MapReduce是个伟大的发明，解决了很多分布式计算问题，但是很遗憾，MapReduce并不是为SQL而设计的。SQL在转换成MapReduce计算原语时，往往需要多层迭代，数据需要较多的落地次数，造成了极大地浪费。

- Impala会尽可能的把数据缓存在内存中，这样数据不落盘即可完成SQL查询，相比MapReduce每一轮迭代都落盘的设计，效率得到极大提升。
- Impala的常驻进程避免了MapReduce启动开销，MapReduce任务的启动开销对于即席查询是个灾难。
- Impala专为SQL而设计，可以避免每次都把任务分解成Mapper和Reducer，减少了迭代的次数，避免了不必要的Shuffle和Sort。

同时Impala现代化的计算框架，能够更好的利用现代的高性能服务器。

- Impala利用LLVM生成动态执行的代码
- Impala会尽可能的利用硬件配置，包括SSE4.1指令集去预取数据等等。
- Impala会自己控制协调磁盘IO，会精细的控制每个磁盘的吞吐，使得总体吞吐最大化。
- 在代码效率层面上，Impala采用C++语言完成，并且追求语言细节，包括内联函数、内循环展开等提速技术
- 在程序内存使用上，Impala利用C++的天然优势，内存占用比JVM系语言小太多，在代码细节层面上也遵循着极少内存使用原则，这使得可以空余出更多的内存给数据缓存。

7) Kudu相比Hbase有何优势，为什么？

Kudu在某些特性上和Hbase很相似，难免会放在一起比较。然而Kudu和Hbase有如下两点本质不同。

- Kudu的数据模型更像是传统的关系型数据库，Hbase是完全的no-sql设计，一切皆是字节。
- Kudu的磁盘存储模型是真正的列式存储，Kudu的存储结构设计和Hbase区别很大。
   综合而言，纯粹的OLTP请求比较适合Hbase，OLTP与OLAP结合的请求适合Kudu。

8) Kudu是纯内存数据库吗？

Kudu不是纯内存数据库，Kudu的数据块分MemRowSet和DiskRowSet，大部分数据存储在磁盘上。

9) Kudu拥有自己的存储格式还是沿用Parquet的？

Kudu的内存存储采用的是行存储，磁盘存储是列存储，其格式和Parquet很相似，部分不相同的部分是为了支持随机读写请求。

10) compactions需要手动操作吗？

compactions被设计为Kudu自动后台执行，并且是缓慢分块执行，当前不支持手动操作。

11) Kudu支持过期自动删除吗?

不支持。Hbase支持该特性。

12) Kudu有和Hbase一样的局部热点问题吗？

现代的分布式存储设计往往会把数据按主键进行有序存储。这样会造成一些局部的热点访问，比如把时间作为主键的日志实时存储模型中，日志的写入总是在时间排序的最后，这在Hbase中会造成严重的局部热点。Kudu也有同样的问题，但是比Hbase好很多，Kudu支持hash分片，数据的写入会先按照hash找到对应的tablet，再按主键有序的写入

13) Kudu在CAP理论中的位置？

和Hbase一样，Kudu是CAP中的CP。只要一个客户端写入数据成功，其他客户端读到的数据都是一致的，如果发生宕机，数据的写入会有一定的延时。

14) Kudu支持多个索引吗？

不支持，Kudu只支持Primary Key一个索引，但是可以把Primary Key设置为包含多列。自动增加的索引、多索引支持、外键等传统数据库支持的特性Kudu正在设计和开发中。

15) Kudu对事务的支持如何？

Kudu不支持多行的事务操作，不支持回滚事务，不过Kudu可以保证单行操作的原子性。







参考：https://blog.csdn.net/qq_35741557/article/details/82690607