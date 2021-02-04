# ClickHouse Copier

## 使用场景

- 集群扩缩容重平衡  
可以使用copier在集群扩容后进行数据重平衡。减少了自己开发重平衡工具的工作量，且效率高效。但是会依赖ZK，增大集群内的网络流量，需要按计划按一定配额进行数据重平衡。

- 集群间数据迁移  
直接使用copier进行集群间数据迁移，也可实现重平衡的功能。

## 使用方法

### schema.xml
任务配置文件，所有的迁移任务包括配置需要在此文件中指定，最后将此文件上传至zookeeper。

文件示例：
```
<yandex>

<remote_servers>
    <!-- 源集群 -->
    <source_cluster>

        <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.230</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

        <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.231</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

    </source_cluster>

    <!-- 目的集群 -->
    <target_cluster>

         <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.230</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

        <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.231</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

        <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.232</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

        <shard>
            <weight>1</weight>
            <replica>
                <host>10.21.130.233</host>
                <port>9002</port>
                <user>default</user>
                <password>default</password>
            </replica>
        </shard>

    </target_cluster>

</remote_servers>

<!-- How many simultaneously active workers are possible. If you run more workers superfluous workers will sleep. -->
<max_workers>1</max_workers>

<tables>
    <!-- A table task, copies tables. -->
    <table_events>

        <cluster_pull>source_cluster</cluster_pull>
        <database_pull>test</database_pull>
        <table_pull>copy_test</table_pull>
        <cluster_push>target_cluster</cluster_push>
        <database_push>copy</database_push>
        <table_push>copy_test</table_push>
        <engine>Engine=MergeTree() ORDER BY c1 PARTITION by pt_dt</engine>
        <sharding_key>rand()</sharding_key>
        <!-- Optional expression that filter data while pull them from source servers -->
        <!-- <where_condition>CounterID != 0</where_condition> -->
        <!-- 选择分区. 可选参数-->
        <!-- <enabled_partitions>
            <partition>'2018-02-26'</partition>
            <partition>'2018-03-05'</partition>
        </enabled_partitions> -->
    </table_events>

</tables>

</yandex>
```

将schema.xml上传到zookeeper上，注意需要现在zk上把路径建好：
```
zkCli.sh create /clickhouse/copytasks/task1
zkCli.sh create /clickhouse/copytasks/task1/description "`cat schema.xml`"
```

### zookeeper.xml

zookeeper配置文件，clickhouse-copier需要使用此文件与zk交互。
配置文件示例:

```
<yandex>
    <logger>
        <level>trace</level>
        <size>1024M</size>
        <count>3</count>
        <log>/data1/clickhouse-copier/copier.log</log>
        <stderr>/data1/clickhouse-copier/stderr.log</stderr>
        <stdout>/data1/clickhouse-copier/stdout.log</stdout>
    </logger>
    <zookeeper>
        <node index="1">
            <host>10.21.130.231</host>
            <port>2181</port>
        </node>
        <node index="1">
            <host>10.21.130.232</host>
            <port>2181</port>
        </node>
        <node index="1">
            <host>10.21.130.233</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```
此配置指定了log配置和zk配置。

### 启动任务

```
clickhouse-copier --config-file=zookeeper.xml --task-path=/clickhouse/copytasks/task1
```

查看日志，如无报错即成功，可以手动校验原表和目的表行数是否一致进行验证数据是否丢失。
