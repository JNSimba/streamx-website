---
id: 'Clickhouse-Connector'
name: 'Clickhouse Connector'
original: true
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

[ClickHouse](https://clickhouse.com/)是一个用于联机分析(OLAP)的列式数据库管理系统(DBMS)，主要面向OLAP场景。目前flink官方未提供写入
读取clickhouse数据的连接器。Streamx 基于ClickHouse 支持的访问形式[HTTP客户端](https://clickhouse.com/docs/zh/interfaces/http/)、
[JDBC驱动](https://clickhouse.com/docs/zh/interfaces/jdbc/)封装了ClickHouseSink用于向clickhouse实时写入数据。

`ClickHouse`写入不支持事务，使用 JDBC 向其中写入数据可提供 AT_LEAST_ONCE (至少一次)的处理语义。使用 HTTP客户端 异步写入，对异步写入重试多次
失败的数据会写入外部组件（kafka,mysql,hdfs,hbase）,最终通过人为介入来恢复数据。

## jdbc同步写入依赖
[ClickHouse](https://clickhouse.com/)提供了[JDBC驱动](https://clickhouse.com/docs/zh/interfaces/jdbc/)
```xml
<dependency>
    <groupId>ru.yandex.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.3.1</version>
</dependency>
```

常规方式下创建clickhouse jdbc连接的方式如下:

<TabItem value="Java" label="Java">

```java
package com.lei.util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;


public class ClickHouseUtil {
    private static Connection connection;

    public static Connection getConn(String host, int port, String database) throws SQLException, ClassNotFoundException {
        Class.forName("ru.yandex.clickhouse.ClickHouseDriver");
        String  address = "jdbc:clickhouse://" + host + ":" + port + "/" + database;
        connection = DriverManager.getConnection(address);
        return connection;
    }

    public static Connection getConn(String host, int port) throws SQLException, ClassNotFoundException {
        return getConn(host,port,"default");
    }
    public static Connection getConn() throws SQLException, ClassNotFoundException {
        return getConn("node-01",8123);
    }
    public void close() throws SQLException {
        connection.close();
    }
}
```
</TabItem>


以上将各项参数拼接为请求 url 的方式较繁琐，并且是硬编码的方式写死的,非常的不灵敏,用`StreamX`接入 `clickhouse`的数据,
只需要按照规定的格式定义好配置文件然后编写代码即可,配置和代码如下

## Jdbc同步写入信息配置

在`StreamX`中`clickhose jdbc` 约定的配置见配置列表，运行程序样例为scala，如下:

<Tabs>
<TabItem value="配置" default>

```yaml
clickhouse:
  sink:
    #    写入节点地址
    jdbcUrl: jdbc:clickhouse://127.0.0.1:8123,192.168.1.2:8123
    socketTimeout: 3000000
    database: test
    user: $user
    password: $password
    #    写入结果表及对应的字段，全部可不指定字段
    targetTable: orders(userId,siteId)
    batch:
      size: 1000
      delaytime: 6000000
```

</TabItem>
<TabItem value="Scala" label="Scala">

```scala
import com.streamxhub.streamx.flink.core.scala.FlinkStreaming
import com.streamxhub.streamx.flink.core.scala.sink.ClickHouseSink
import org.apache.flink.api.scala._

object ClickHouseSinkApp extends FlinkStreaming {

  override def handle(): Unit = {
    val createTable =
      """
        |create TABLE test.orders(
        |userId UInt16,
        |orderId UInt16,
        |siteId UInt8,
        |cityId UInt8,
        |orderStatus UInt8,
        |price Float64,
        |quantity UInt8,
        |timestamp UInt16
        |)ENGINE = TinyLog;
        |""".stripMargin

    println(createTable)

    val source = context.addSource(new TestSource)


    // jdbc同步写入写入
       ClickHouseSink().syncSink[TestEntity](source)(x => {
         s"(${x.userId},${x.siteId})"
       }).setParallelism(1)
  }

}

class Order(val marketId: String, val timestamp: String) extends Serializable
```
</TabItem>
</Tabs>

:::tip 提示
clickhouse 可支持多个节点均衡写入，只需在jdbcUrl配置可写入的节点即可<br></br>
由于ClickHouse单次插入的延迟比较高，建议设置 batch.size 来批量插入数据提高性能,同时为了提高实时性，
支持按照数据量或者间隔时间 batch.delaytime 来批次写入。<br></br>
在ClickHouseSink的实现中，若最后一批数据的数目不足BatchSize，则会在关闭连接时候插入剩余数据。
:::

## http异步写入依赖
```xml
        <!--clickhouse async need asynchttpclient -->
        <dependency>
            <groupId>org.asynchttpclient</groupId>
            <artifactId>async-http-client</artifactId>
            <optional>true</optional>
        </dependency>
```

clickhouse INSERT 必须通过POST方法来插入数据 常规操作如下：

```bash
$ echo 'INSERT INTO t VALUES (1),(2),(3)' | curl 'http://localhost:8123/' --data-binary @-
```
上述方式操作较简陋，streamx 对 http post 写入方式进行封装增强，增加缓存、异步写入、失败重试、达到重试阈值后数据备份至外部组件（kafka,mysql,hdfs,hbase）
等功能，以上功能只需要按照规定的格式定义好配置文件然后编写代码即可,配置和代码如下

## http异步写入信息配置

在`StreamX`中`clickhose jdbc` 约定的配置见配置列表，运行程序样例为scala，如下:

<Tabs>
<TabItem value="配置" default>

```yaml

clickhouse:
  sink:
    hosts: 127.0.0.1:8123,192.168.1.2:8123
    socketTimeout: 3000000
    database: test
    user: $user
    password: $password
    targetTable: test.orders(userId,siteId)
    batch:
      size: 1
      delaytime: 60000
    threshold:
      bufferSize: 10
      #      异步写入的并发数
      numWriters: 4
      #      缓存队列大小
      queueCapacity: 100
      delayTime: 10
      requestTimeout: 600
      retries: 1
      #      成功响应码
      successCode: 200
    failover:
      table: chfailover
      #      达到失败最大写入次数后，数据备份的组件
      storage: kafka #kafka|mysql|hbase|hdfs
      mysql:
        driverClassName: com.mysql.cj.jdbc.Driver
        jdbcUrl: jdbc:mysql://10.10.130.220:3306/test?useSSL=false&allowPublicKeyRetrieval=true
        username: $user
        password: $pass
      kafka:
        bootstrap.servers: slave3.test.gitv.we:9092,slave4.test.gitv.we:9092,slave2.test.gitv.we:9092
        topic: test1
        group.id: user_01
        auto.offset.reset: latest
      hbase:
        zookeeper.quorum: localhost
        zookeeper.property.clientPort: 2181 
      hdfs:
        path: /data/chfailover
        namenode: hdfs://master2.test.gitv.we:8020
        user: hdfs
```

</TabItem>
<TabItem value="Scala" label="Scala">

```scala
import com.streamxhub.streamx.flink.core.scala.FlinkStreaming
import com.streamxhub.streamx.flink.core.scala.sink.ClickHouseSink
import org.apache.flink.api.scala._

object ClickHouseSinkApp extends FlinkStreaming {

  override def handle(): Unit = {
    val createTable =
      """
        |create TABLE test.orders(
        |userId UInt16,
        |orderId UInt16,
        |siteId UInt8,
        |cityId UInt8,
        |orderStatus UInt8,
        |price Float64,
        |quantity UInt8,
        |timestamp UInt16
        |)ENGINE = TinyLog;
        |""".stripMargin

    println(createTable)

    val source = context.addSource(new TestSource)


    // 异步写入
    ClickHouseSink().sink[TestEntity](source)(x => {
      s"(${x.userId},${x.siteId})"
    }).setParallelism(1)

  }

}

class Order(val marketId: String, val timestamp: String) extends Serializable
```
</TabItem>
</Tabs>

:::info 警告
由于ClickHouse单次插入的延迟比较高，小数据量频繁写入会造成clickhouse server 频繁排序合并分区，建议使用异步提交方式，设置合理阈值提高性能<br></br>
由于ClickHouse 异步写入失败会重新将数据添加至缓存队列，可能造成同一窗口数据分两批次写入，实时性要求高的场景建议全面测试clickhouse的稳定性<br></br>
异步写入数据达到重试最大值后，会将数据备份至外部组件，在此时才会初始化组件连接，建议确保failover 组件的可用性
:::

## 其他配置

其他的所有的配置都必须遵守 ** ClickHouseDataSource ** 连接池的配置,具体可配置项和各个参数的作用请参考`光 clickhouse-jdbc`[官网文档](https://github.com/ClickHouse/clickhouse-jdbc).