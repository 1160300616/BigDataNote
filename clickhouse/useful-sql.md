# ClickHouse useful SQL

## 查看库表容量，压缩率等
```
select database,
　　table,
　　formatReadableSize(sum(bytes)) as size,
　　sum(rows) as rows,
　　formatReadableSize(sum(bytes_on_disk)) as bytes_on_disk,
　　formatReadableSize(sum(data_uncompressed_bytes)) as data_uncompressed_bytes,
　　formatReadableSize(sum(data_compressed_bytes)) as data_compressed_bytes

from system.parts
where active
　　and database = 'behavior'
　　and table = 'test_local'
　　group by database, table
```

## 查询clickhouse支持的数据类型
```
select * from  system.data_type_families 
```

## 查看支持的format格式
```
select * from   system.formats 
```

## 创建分布式表
```
create table migrate_test_v1 as migrate_test_v1_local 
ENGINE = Distributed('cluster-test', test, migrate_test_v1_local, rand())
```

## 创建复制表

在配置文件config.xml中指定zookeeper配置：
```
<default_replica_path>/clickhouse/tables/{shard}/{database}/{table}</default_replica_path>
<default_replica_name>{replica}</default_replica_name>
```
以下两条SQL等价：
```
CREATE TABLE table_name (
x UInt32
) ENGINE = ReplicatedMergeTree
ORDER BY x
```
```
CREATE TABLE table_name (
x UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}')
ORDER BY x
```
## 查看某个用户最近的查询SQL
```
select query,user,query_start_time,query_duration_ms from system.query_log where user = 'default' order by query_start_time desc limit 100 
```
## 远程表
```
select * from remote('目标IP',db.table,'user','passwd')
```
通过remote我们可以在一个节点访问另一个节点上的数据，使用比较方便灵活。

## AggregatingMergeTree 结合物化视图
- 建表
```
CREATE MATERIALIZED VIEW test.basic
ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(StartDate) ORDER BY (CounterID, StartDate)
AS SELECT
    CounterID,
    StartDate,
    sumState(Sign)    AS Visits,
    uniqState(UserID) AS Users
FROM test.visits
GROUP BY CounterID, StartDate;
```
注意，在这里使用到了AggregateFunction，存储中间状态需要在聚合函数后面加上State
- 查询
```
SELECT
    StartDate,
    sumMerge(Visits) AS Visits,
    uniqMerge(Users) AS Users
FROM test.basic
GROUP BY StartDate
ORDER BY StartDate;
```
注意，在这里使用到了AggregateFunction，获得最终结果需要在聚合函数后面加上Merge

## 权限控制
经过测试，我们可以通过JDBC、客户端创建用户

- 建立用户
```
create user user_test on cluster 'cluster-test' IDENTIFIED  with plaintext_password BY 'test'  
```
- 建立角色
```
CREATE ROLE r_o on cluster 'cluster-test'
```
- 授权table表上x和y字段的select权限
```
GRANT SELECT(x,y) ON db.table TO john WITH GRANT OPTION  
```
- db上所有表的select权限
```
GRANT SELECT ON db.* TO john 
```
- 当前数据库上所有表的select权限
```
GRANT SELECT ON * TO john  
```

- 所有库所有表上的select和insert权限
```
GRANT SELECT, INSERT ON *.* TO john, robin 
```
