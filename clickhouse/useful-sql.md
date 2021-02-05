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

## 权限控制
经过测试，我们可以通过JDBC、客户端、HTTP创建用户

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
