# ClickHouse Better Practices

## 表相关事项

### 数据类型

- 建表时能用数值型或日期时间型表示的字段，就不要用字符串——全String类型在以Hive为中心的数仓建设中常见，但CK环境不应受此影响。
- 直接用DateTime表示时间列，而不是用整形的时间戳。因为CK中DateTime的底层就是时间戳，效率高，可读性好，且转换函数丰富。
- 官方已经指出Nullable类型几乎总是会拖累性能，因为存储Nullable列时需要创建一个额外的文件来存储NULL的标记，并且Nullable列无法被索引。因此除非极特殊情况，应直接使用字段默认值表示空，或者自行指定一个在业务中无意义的值（例如用-1表示没有商品ID）。

### 分区和索引
- 事实表必须分区，分区粒度根据业务特点决定，不宜过粗或过细。按天分区，按小时、周、月分区也比较常见（系统表中的query_log、trace_log表默认就是按月分区的）。


### 表参数
- 生产环境中提供线上服务的表均采用复制表与分布式表相结合.表名命名规范
- 如果表中不是必须保留全量历史数据，建议指定TTL
- 建议指定use_minimalistic_part_header_in_zookeeper = 1设置项，能够显著压缩表元数据在ZooKeeper中的存储。该项也可以写入config.xml中的<merge_tree>一节。

## 查询相关事项

### 单表查询
- 所有应用层查询禁止SELECT *
- 查询分区表必须指定分区（所谓partition pruning），不能全表查询
- 业务场景非强制要求100%准确的基数计量，应该用uniq()函数而不是uniqExact()函数或DISTINCT关键字。uniq()底层采用HyperLogLog实现，能够以低于1%的精度损失换来极大的性能提升。
- 能够重用的模式化查询（如固定刷新的BI报表、热力图等）一定要做成物化视图，并在物化视图上查询出结果，可以避免大量的重复计算。


### 多表查询
- 当两表关联查询只需要从左表出结果时，建议用IN而不是JOIN，即写成SELECT ... FROM left_table WHERE join_key IN (SELECT ... FROM right_table)的风格。
- 不管是LEFT、RIGHT还是INNER JOIN操作，小表都必须放在右侧。因为CK默认在大多数情况下都用hash join算法，左表固定为probe table，右表固定为build table且被广播。
- CK的查询优化器比较弱，JOIN操作的谓词不会下推，因此一定要先做完过滤、聚合等操作，再在结果集上做JOIN。这点与我们写其他平台SQL语句的习惯很不同，初期尤其需要注意
- 两张分布式表上的IN和JOIN之前必须加上GLOBAL关键字

## 写入相关事项
- 写入分布式表的底表，而不直接写分布式表。
- 不要做小批量零碎的写入，每批次至少千条级别，避免给merge造成太大压力。
- 不要同时写入太多个分区，或者写入过快（官方给出的阈值为1秒1次），容易因为merge的速度跟不上parts生成的速度而报出"too many parts"的错误。

## 运维相关事项

### CPU
CK的“快”与其对CPU的积极利用密不可分，所以CPU的单核性能和多核性能都要尽量好一点，16核32线程左右且带较高的睿频比较合适。CK设置中的max_threads参数控制单个查询所能利用的CPU线程数，默认与本机CPU的物理核心数相同，如果服务器是CK独占的，那么就不用改，否则就改小些。

在监控集群时，CPU指标也是最重要的。实测当单个CK Server节点的CPU使用率超过70%时，服务就不太稳定了。

### 内存
官方文档建议单机物理内存128G左右。实测CK在我们的应用场景下内存占用并不激进，每线程对应1G内存非常绰绰有余，即max_threads设为20的话，max_memory_usage参数设为20G（懒得打辣么多0了）。为了不干扰系统的正常运行，也应配置所有查询能利用的最大内存参数max_memory_usage_for_all_queries，取物理内存的80%左右即可。

另外，CK在执行GROUP BY聚合逻辑的过程中很有可能超出内存限制，因此也建议设置max_bytes_before_external_group_by参数。在内存占用超出此阈值之后，就会spill到磁盘继续操作，且性能没有降低特别多。官方建议将它设置为max_memory_usage的一半。

### 存储
为了快速响应，或者多数查询的数据量都很大，建议上SSD

### ZooKeeper
千万要调教好ZooKeeper集群，一旦ZK不可用，复制表和分布式表就不可用了。ZK的数据量基本上与CK的数据量成正相关，所以一定要配置自动清理
```
autopurge.purgeInterval = 1
autopurge.snapRetainCount = 5
```
