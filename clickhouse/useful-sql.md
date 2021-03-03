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
## 查看最近的查询SQL
```
SELECT   
    event_time,   
    user,   
    query_id AS query,   
    read_rows,   
    read_bytes,   
    result_rows,   
    result_bytes,   
    memory_usage,   
    exception  
FROM cluster('replica-1', system, query_log)  
WHERE (event_date = today()) AND (event_time >= (now() - 60)) AND (is_initial_query = 1) AND (query NOT LIKE 'INSERT INTO%')  
ORDER BY event_time DESC  
LIMIT 100  
```
## 查看慢查询
```
SELECT   
    event_time,   
    user,   
    query_id as query,  
    query_duration_ms/1000,
    read_rows,   
    read_bytes,   
    result_rows,   
    result_bytes,   
    memory_usage,   
    exception  
FROM cluster('replica-1', system, query_log)  
WHERE (event_date = yesterday()) AND query_duration_ms > 30000 AND (is_initial_query = 1) AND (query NOT LIKE 'INSERT INTO%')  
ORDER BY query_duration_ms desc  
LIMIT 100  
```

## 查看top 10 大表(聚合结果)
```
SELECT   
    database,   
    table,   
    sum(bytes_on_disk) AS bytes_on_disk  
FROM cluster('replica-1', system, parts)  
WHERE active AND (database != 'system')  
GROUP BY   
    database,   
    table  
ORDER BY bytes_on_disk DESC  
LIMIT 10
```

## 查看top 10 大表（非聚合）
```
SELECT   
    database,   
    table,   
    hostname() as host,
    sum(bytes_on_disk) AS bytes_on_disk  
FROM cluster('replica-1', system, parts)  
WHERE active AND (database != 'system')  
GROUP BY   
    database,   
    table  ,
    host 
ORDER BY bytes_on_disk DESC
LIMIT 10
```
## 查看top10 查询用户
```
SELECT   
    user,   
    count(1) AS query_times,   
    sum(read_bytes) AS query_bytes,   
    sum(read_rows) AS query_rows  
FROM cluster('replica-1', system, query_log)  
WHERE (event_date = yesterday()) AND (is_initial_query = 1) AND (query NOT LIKE 'INSERT INTO%')  
GROUP BY user  
ORDER BY query_times DESC  
LIMIT 10
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

## 查看SQL执行计划

```
EXPLAIN SELECT * FROM TEST.TEST1
```
新版本clickhouse支持查看SQL执行计划，通过使用EXPLAIN关键字。

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
GRANT ON CLUSTER 'cluster-test' SELECT ON db.* TO john 
```
- 当前数据库上所有表的select权限
```
GRANT SELECT ON * TO john  
```

- 所有库所有表上的select和insert权限
```
GRANT SELECT, INSERT ON *.* TO john, robin 
```
